# Guía: Deploy de nuevo microservicio en GestorCooperativo

Basada en aprendizajes del deploy de `svc-vivienda` (jun 2026).
Seguir este orden evita los errores ya encontrados.

---

## Prerequisitos (ya hecho una vez, no repetir)

- ✅ Proyecto GCP `gestorcooperativo` con billing vinculado
- ✅ Cloud SQL instancia `ministerio-postgres` corriendo
- ✅ Artifact Registry `ministerio-docker` creado
- ✅ API Gateway `ministerio-gateway` desplegado en `us-central1`
- ✅ Firebase Authentication configurado (providers: Google + Email/Password)
- ✅ Cloud Build trigger configurado para `panel.backend`

---

## Checklist por nuevo servicio (ej: `svc-gasifera`)

### 1. Reunión de área y spec aprobado
- [ ] Completar `docs/context/areas/{area}/contexto_detallado.md`
- [ ] Spec en `docs/files/spec-svc-{nombre}.md` → estado `approved`

### 2. Scaffold del servicio (copiar estructura de svc-vivienda)

```
services/svc-{nombre}/
├── Dockerfile
├── pyproject.toml       ← ojo: incluir [tool.hatch.build.targets.wheel] packages = ["app"]
├── alembic.ini
├── alembic/versions/
└── app/
    ├── __init__.py
    ├── main.py
    ├── config.py        ← usar Firebase como issuer (ver template)
    ├── auth.py          ← copiar de svc-vivienda (leer X-Forwarded-Authorization)
    ├── database.py
    └── {modulo}/
```

**config.py template:**
```python
google_jwks_uri: str = "https://www.googleapis.com/service_accounts/v1/jwk/securetoken@system.gserviceaccount.com"
google_issuer: str = "https://securetoken.google.com/gestorcooperativo"
```

**Dockerfile CMD obligatorio:**
```dockerfile
CMD ["sh", "-c", "uvicorn app.main:app --host 0.0.0.0 --port ${PORT:-8080} --workers 1"]
```

### 3. GCP: service account + secret + base de datos

```bash
# Service account
gcloud iam service-accounts create svc-{nombre} \
  --display-name="Servicio {nombre}" --project=gestorcooperativo

# Roles mínimos
for role in roles/cloudsql.client roles/secretmanager.secretAccessor roles/pubsub.publisher; do
  gcloud projects add-iam-policy-binding gestorcooperativo \
    --member="serviceAccount:svc-{nombre}@gestorcooperativo.iam.gserviceaccount.com" \
    --role="$role"
done

# Base de datos en Cloud SQL
gcloud sql databases create db_{nombre} \
  --instance=ministerio-postgres --project=gestorcooperativo

# Usuario DB
gcloud sql users create user_{nombre} \
  --instance=ministerio-postgres \
  --password="{GENERAR_PASSWORD_SEGURA}" --project=gestorcooperativo

# Secret con URL de conexión (formato socket para Cloud Run)
echo -n "postgresql+asyncpg://user_{nombre}:{PASSWORD}@/db_{nombre}?host=/cloudsql/gestorcooperativo:southamerica-east1:ministerio-postgres" | \
  gcloud secrets create svc-{nombre}-db-url \
  --data-file=- --project=gestorcooperativo

gcloud secrets add-iam-policy-binding svc-{nombre}-db-url \
  --member="serviceAccount:svc-{nombre}@gestorcooperativo.iam.gserviceaccount.com" \
  --role="roles/secretmanager.secretAccessor" --project=gestorcooperativo
```

### 4. Permisos para API Gateway

```bash
# Invoker en el nuevo Cloud Run service (se ejecuta DESPUÉS del primer deploy)
gcloud run services add-iam-policy-binding svc-{nombre} \
  --member="serviceAccount:api-gateway-sa@gestorcooperativo.iam.gserviceaccount.com" \
  --role="roles/run.invoker" \
  --region=southamerica-east1 --project=gestorcooperativo
```

> **Nota:** `roles/iam.serviceAccountTokenCreator` para el service agent ya fue otorgado al desplegar svc-vivienda. No repetir a menos que se cree un nuevo gateway.

### 5. Alembic: primera migración

```bash
# Desde Cloud Shell con proxy corriendo
cd ~/gestorcooperativo/backend/svc-{nombre}
pip install --user -e ".[dev]"

# URL TCP (proxy local) — distinta al secret (que usa socket)
PASSWORD=$(gcloud secrets versions access latest \
  --secret=svc-{nombre}-db-url | sed 's|.*://[^:]*:\([^@]*\)@.*|\1|')
export DATABASE_URL="postgresql+asyncpg://user_{nombre}:${PASSWORD}@127.0.0.1:5432/db_{nombre}"

~/.local/bin/alembic upgrade head
```

### 6. Actualizar openapi.yaml del gateway

Agregar las rutas del nuevo servicio en `infra/gateway/openapi.yaml`:
```yaml
/api/v1/{nombre}/{path+}:
  get:
    operationId: listar{Nombre}
    x-google-backend:
      address: "https://svc-{nombre}-HASH-rj.a.run.app"  # URL real post-deploy
      path_translation: APPEND_PATH_TO_ADDRESS
      jwt_audience: "https://svc-{nombre}-HASH-rj.a.run.app"
```

Incrementar `API_CONFIG_ID` en `deploy-gateway.sh` (ej: `v4`) y re-ejecutar.

> ⚠️ Todo endpoint necesita `x-google-backend`. Sin excepción.

### 7. Primer deploy vía Cloud Build

```bash
# Desde services/ (raíz del repo panel.backend)
gcloud builds submit \
  --config=cloudbuild.yaml \
  --substitutions=_SERVICE=svc-{nombre} \
  --project=gestorcooperativo
```

O simplemente push a `main` (el trigger lo hace automáticamente si el path filter está configurado).

### 8. Verificación post-deploy

```bash
# Health check
curl -s https://svc-{nombre}-HASH-rj.a.run.app/health

# Via gateway (requiere token Firebase)
FIREBASE_API_KEY="AIzaSyBvim_XuWaXyMMjAIyVRy9QU3OG2Mgo9NQ"
TOKEN=$(curl -s -X POST \
  "https://identitytoolkit.googleapis.com/v1/accounts:signInWithPassword?key=${FIREBASE_API_KEY}" \
  -H "Content-Type: application/json" \
  -d '{"email":"test.operador@ministerio-test.com","password":"Test1234!","returnSecureToken":true}' \
  | python3 -c "import sys,json; print(json.load(sys.stdin).get('idToken',''))")

curl -s -H "Authorization: Bearer $TOKEN" \
  https://ministerio-gateway-3j5k00ma.uc.gateway.dev/api/v1/{nombre}/health
```

---

## Errores frecuentes y soluciones rápidas

| Error | Causa | Fix |
|-------|-------|-----|
| `ModuleNotFoundError` en startup | Falta `[tool.hatch.build.targets.wheel]` | Agregar `packages = ["app"]` en pyproject.toml |
| Puerto incorrecto en Cloud Run | Puerto hardcodeado | `${PORT:-8080}` en Dockerfile CMD |
| Cloud Run 403 desde gateway | SA sin `roles/run.invoker` | `gcloud run services add-iam-policy-binding ...` |
| Gateway 403 en todos los endpoints | SA sin `serviceAccountTokenCreator` | Otorgar al service agent de API Gateway |
| `ALREADY_EXISTS` en api-configs | Config inmutable | Incrementar versión: `-v2`, `-v3`... |
| `Jwt issuer is not configured` | Issuer en security definition no matchea | Verificar `iss` del token y ajustar `x-google-issuer` |
| `AUTH_TOKEN_INVALIDO` en backend | Backend lee `Authorization` en lugar de `X-Forwarded-Authorization` | Usar patrón de `auth.py` de svc-vivienda |
| Alembic `Path doesn't exist` | Ejecutar desde directorio equivocado | `cd svc-{nombre}/` antes de `alembic upgrade head` |
| IAM 403 inmediato después de grant | Propagation delay | Esperar 2 minutos |
