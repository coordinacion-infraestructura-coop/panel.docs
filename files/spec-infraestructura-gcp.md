# Spec: infraestructura GCP y API Gateway

**Estado**: draft | **Versión**: 0.2.0

---

## 1. Proyecto GCP

```
Project ID:   gestorcooperativo
Region:       southamerica-east1   (São Paulo — menor latencia desde Córdoba, AR)
```

## 2. API Gateway — spec OpenAPI base

Archivo a crear: `svc-gateway/openapi.yaml`

```yaml
swagger: "2.0"
info:
  title: Sistema Secretaría General de Gobierno (ex Ministerio de Cooperativas y Mutuales)
  description: API Gateway unificado para todos los subsistemas de la secretaría
  version: "1.0.0"
host: api.ministerio-coop.gob.ar
schemes:
  - https
produces:
  - application/json

securityDefinitions:
  google:
    authorizationUrl: ""
    flow: implicit
    type: oauth2
    x-google-issuer: "https://accounts.google.com"
    x-google-jwks_uri: "https://www.googleapis.com/oauth2/v3/certs"
    x-google-audiences: "gestorcooperativo"

security:
  - google: []

paths:
  # --- svc-privada (sistema existente — proyecto GCP: essential-haiku-482815-u4) ---
  /api/v1/privada/gestiones:
    get:
      operationId: listarGestionesPrivada
      summary: Listado de gestiones privadas del ministro
      x-google-backend:
        address: https://infraestructura-gestioninterna-354063050046.southamerica-east1.run.app
        path_translation: APPEND_PATH_TO_ADDRESS
      responses:
        "200":
          description: Lista de gestiones
    post:
      operationId: crearGestionPrivada
      x-google-backend:
        address: https://infraestructura-gestioninterna-354063050046.southamerica-east1.run.app
        path_translation: APPEND_PATH_TO_ADDRESS
      responses:
        "201":
          description: Gestión creada

  /api/v1/privada/{resource_path}:
    get:
      operationId: proxyGetPrivada
      parameters:
        - in: path
          name: resource_path
          required: true
          type: string
      x-google-backend:
        address: https://infraestructura-gestioninterna-354063050046.southamerica-east1.run.app
        path_translation: APPEND_PATH_TO_ADDRESS
      responses:
        "200":
          description: Respuesta del svc-privada

  # --- svc-vivienda ---
  /api/v1/vivienda/beneficiarios:
    get:
      operationId: listarBeneficiariosVivienda
      x-google-backend:
        address: https://svc-vivienda-HASH-southamerica-east1.a.run.app
      responses:
        "200":
          description: Lista de beneficiarios
    post:
      operationId: crearBeneficiarioVivienda
      x-google-backend:
        address: https://svc-vivienda-HASH-southamerica-east1.a.run.app
      responses:
        "201":
          description: Beneficiario creado

  # --- svc-infraestructura ---
  /api/v1/infraestructura/proyectos:
    get:
      operationId: listarProyectosInfraestructura
      x-google-backend:
        address: https://svc-infraestructura-HASH-southamerica-east1.a.run.app
      responses:
        "200":
          description: Lista de proyectos

  # --- svc-territorial ---
  /api/v1/territorial/cooperativas:
    get:
      operationId: listarCooperativasTerritorial
      x-google-backend:
        address: https://svc-territorial-HASH-southamerica-east1.a.run.app
      responses:
        "200":
          description: Lista de cooperativas

  # --- svc-desarrollo ---
  /api/v1/desarrollo/cooperativas:
    get:
      operationId: listarCooperativasDesarrollo
      x-google-backend:
        address: https://svc-desarrollo-HASH-southamerica-east1.a.run.app
      responses:
        "200":
          description: Lista de cooperativas (UTN)

  # --- svc-gasifera ---
  /api/v1/gasifera/creditos:
    get:
      operationId: listarCreditosGasifera
      x-google-backend:
        address: https://svc-gasifera-HASH-southamerica-east1.a.run.app
      responses:
        "200":
          description: Lista de créditos
```

> Nota: las URLs `HASH-southamerica-east1.a.run.app` se reemplazan con las URLs reales de Cloud Run al deployar. Se recomienda usar variables en el pipeline CI/CD.

---

## 3. Cloud Run — configuración por servicio

Todos los servicios comparten esta configuración base:

```yaml
# cloudbuild.yaml (base)
steps:
  - name: 'gcr.io/cloud-builders/docker'
    args: ['build', '-t', 'southamerica-east1-docker.pkg.dev/gestorcooperativo/${_SERVICE_NAME}:${SHORT_SHA}', '.']
  - name: 'gcr.io/cloud-builders/docker'
    args: ['push', 'southamerica-east1-docker.pkg.dev/gestorcooperativo/${_SERVICE_NAME}:${SHORT_SHA}']
  - name: 'gcr.io/google.com/cloudsdktool/cloud-sdk'
    entrypoint: gcloud
    args:
      - run
      - deploy
      - ${_SERVICE_NAME}
      - --image=southamerica-east1-docker.pkg.dev/gestorcooperativo/${_SERVICE_NAME}:${SHORT_SHA}
      - --region=southamerica-east1
      - --platform=managed
      - --no-allow-unauthenticated
      - --service-account=${_SERVICE_NAME}@gestorcooperativo.iam.gserviceaccount.com
      - --set-secrets=DATABASE_URL=${_SERVICE_NAME}-db-url:latest
      - --memory=512Mi
      - --cpu=1
      - --min-instances=0
      - --max-instances=10
      - --concurrency=80
```

Parámetros `--no-allow-unauthenticated` es crítico: los servicios Cloud Run no son públicos, solo reciben tráfico del API Gateway.

---

## 4. Service accounts

Un service account por servicio (principio de mínimo privilegio):

| Service Account | Permisos |
|-----------------|----------|
| `svc-vivienda@...` | Cloud SQL Client, Secret Manager Accessor, Pub/Sub Publisher |
| `svc-infraestructura@...` | Cloud SQL Client, Secret Manager Accessor, Pub/Sub Publisher |
| `svc-territorial@...` | Cloud SQL Client, Secret Manager Accessor, Pub/Sub Publisher |
| `svc-desarrollo@...` | Cloud SQL Client, Secret Manager Accessor, Pub/Sub Publisher |
| `svc-gasifera@...` | Cloud SQL Client, Secret Manager Accessor, Pub/Sub Publisher |
| `api-gateway@...` | Cloud Run Invoker (todos los servicios) |

---

## 5. Cloud SQL — bases de datos

Una instancia Cloud SQL PostgreSQL 15 con múltiples bases de datos:

```
Instancia: ministerio-postgres (db-f1-micro en dev, db-g1-small en prod)
Bases de datos:
  - db_vivienda
  - db_infraestructura
  - db_territorial
  - db_desarrollo
  - db_gasifera
```

Conexión desde Cloud Run: via Cloud SQL Auth Proxy (socket Unix, sin IP pública).

---

## 6. Secrets en Secret Manager

```
svc-vivienda-db-url         → postgresql+asyncpg://user:pass@/db_vivienda?...
svc-infraestructura-db-url  → postgresql+asyncpg://...
svc-territorial-db-url      → postgresql+asyncpg://...
svc-desarrollo-db-url       → postgresql+asyncpg://...
svc-gasifera-db-url         → postgresql+asyncpg://...
utn-api-key                 → [key provista por UTN]
firebase-admin-sdk-key      → [JSON del SDK admin]
```

---

## 7. Pub/Sub — tópicos

```
ministerio-eventos-vivienda
ministerio-eventos-infraestructura
ministerio-eventos-territorial
ministerio-eventos-desarrollo
ministerio-eventos-gasifera
ministerio-eventos-dl          → dead letter queue
```

Subscripciones:
- `bigquery-sub-*` → cada tópico tiene una suscripción que alimenta BigQuery
- `notificaciones-sub-*` → `svc-notificaciones` escucha eventos que disparan comunicaciones

---

## 8. BigQuery — dataset ejecutivo

```
Dataset: ministerio_ejecutivo
Tablas:
  - vivienda_expedientes_hist
  - vivienda_estadisticas_diarias
  - infraestructura_proyectos_hist
  - territorial_actividades_hist
  - desarrollo_cooperativas_sync
  - gasifera_creditos_hist
  - gasifera_cuotas_hist
```

---

## 9. Firebase Hosting

```
firebase.json:
{
  "hosting": {
    "public": "build",
    "ignore": ["firebase.json", "**/.*", "**/node_modules/**"],
    "rewrites": [{ "source": "**", "destination": "/index.html" }],
    "headers": [
      {
        "source": "/static/**",
        "headers": [{ "key": "Cache-Control", "value": "public, max-age=31536000, immutable" }]
      },
      {
        "source": "/index.html",
        "headers": [{ "key": "Cache-Control", "value": "no-cache, no-store, must-revalidate" }]
      }
    ]
  }
}
```

---

## 10. Criterios de aceptación infraestructura

- [ ] Todos los servicios Cloud Run sin acceso público directo
- [ ] API Gateway validando JWT en todos los endpoints
- [ ] Un service account por servicio con mínimo privilegio
- [ ] Secrets solo accesibles por el servicio correspondiente
- [ ] Tópicos Pub/Sub creados con dead letter queue
- [ ] Pipeline CI/CD funcional para cada servicio
- [ ] Firebase Hosting con dominio custom y HTTPS
- [ ] Cloud Armor habilitado delante del API Gateway
