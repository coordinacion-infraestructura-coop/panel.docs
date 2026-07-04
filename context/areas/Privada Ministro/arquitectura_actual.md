# Secretaría Privada del Ministro — Arquitectura del Sistema Actual

## Estado

Sistema **en producción** desde 2024. No requiere reescritura, se integra al nuevo sistema vía API Gateway.

---

## Stack técnico

| Capa | Tecnología | Detalle |
|------|-----------|---------|
| Backend | FastAPI (Python 3.12) | Cloud Run, región `southamerica-east1` |
| Base de datos | Google BigQuery | Dataset `infra_gestion`, proyecto GCP `essential-haiku-482815-u4` |
| Frontend | Vanilla JS + HTML5 + CSS3 | GitHub Pages |
| Auth | Google OAuth 2.0 | Google Identity Services (accounts.google.com) |
| Analítica | Looker Studio | Embebido como iframe |

## Proyecto GCP del sistema existente

```
Project ID:  essential-haiku-482815-u4
Region:      southamerica-east1 (São Paulo)
```

## URLs de producción

```
Backend API:  https://infraestructura-gestioninterna-354063050046.southamerica-east1.run.app
Frontend:     https://labotech-analytics.github.io/SistemaGestiones_infraestructura_front/
```

---

## Dataset BigQuery: `infra_gestion`

| Tabla | Descripción | Registros (2026-06) |
|-------|------------|---------------------|
| `gestiones` | Central de casos/demandas | ~523 activas |
| `gestiones_eventos` | Auditoría de cambios de estado | — |
| `usuarios_roles` | Control de acceso | — |
| `localidades_info` | Datos enriquecidos por localidad | 551 |
| `departamentos_info` | Datos por departamento | 26 |
| `geo_localidades` | Referencia geoespacial | 551 |
| `cat_estado` | Catálogo de estados | 6 |
| `cat_urgencia` | Catálogo de urgencias | 3 |
| `cat_ministerio_agencia` | Organismos | ~14 |
| `cat_categoria_general` | Categorías de obra | ~20 |
| `cat_tipo_gestion` | Tipos de gestión | variable |
| `cat_canal_origen` | Canales de origen | variable |

Vista analítica: `v_informe_cooperativas` — clasifica gestiones en 10 temas.

---

## Endpoints de la API

| Método | Endpoint | Roles | Descripción |
|--------|---------|-------|-------------|
| GET | `/me` | Todos | Usuario actual `{email, nombre, rol}` |
| GET | `/catalogos/estados` | Todos | Catálogo de estados |
| GET | `/catalogos/urgencias` | Todos | Catálogo de urgencias |
| GET | `/catalogos/ministerios` | Todos | Ministerios/organismos |
| GET | `/catalogos/categorias` | Todos | Categorías |
| GET | `/catalogos/tipos-gestion` | Todos | Tipos de gestión |
| GET | `/catalogos/canales-origen` | Todos | Canales de origen |
| GET | `/catalogos/departamentos` | Todos | Departamentos |
| GET | `/catalogos/localidades?departamento=X` | Todos | Localidades |
| GET | `/gestiones/` | Admin/Supervisor/Operador/Consulta | Listado paginado con filtros |
| GET | `/gestiones/{id}` | Todos | Detalle de gestión |
| GET | `/gestiones/{id}/eventos` | Todos | Historial de cambios |
| POST | `/gestiones/` | Admin/Supervisor/Operador | Crear gestión |
| POST | `/gestiones/{id}/cambiar-estado` | Admin/Supervisor/Operador | Cambio de estado |
| DELETE | `/gestiones/{id}` | Admin/Supervisor | Borrado lógico |
| GET | `/gestiones/resumen-territorial` | Todos | Resumen por departamento/localidad |
| GET | `/localidades-info` | Todos | Datos enriquecidos de localidad |
| PUT | `/localidades-info` | Admin/Supervisor/Operador | Upsert datos localidad |
| GET | `/usuarios/` | Admin | Lista de usuarios |
| POST | `/usuarios/` | Admin | Crear usuario |
| PUT | `/usuarios/{email}` | Admin | Actualizar usuario |
| DELETE | `/usuarios/{email}` | Admin | Desactivar usuario |
| GET | `/informe/cooperativas/resumen` | Todos | KPIs por tema |
| GET | `/informe/cooperativas/temporal` | Todos | Evolución mensual |
| GET | `/informe/cooperativas/por-departamento` | Todos | Por departamento |
| GET | `/informe/cooperativas/puntos` | Todos | Puntos geoespaciales para mapa |

## Roles del sistema existente

| Rol actual | Equivalente propuesto rol ministerial | Notas |
|-----------|--------------------------------------|-------|
| Admin | `admin_sistema` o `secretario` | Definir en reunión con el área |
| Supervisor | `director` | — |
| Operador | `operador` | — |
| Consulta | `ministro` (solo lectura) | — |

> **Pendiente**: reunión con Secretaría Privada del Ministro para confirmar el mapeo de roles y definir si los campos actuales del sistema cubren todas las necesidades del área.

---

## Integración con el nuevo sistema (ADR-006)

El sistema existente se expone al nuevo API Gateway sin migración:

```
Frontend ministerial
    ↓
API Gateway gestorcooperativo (proyecto nuevo)
    ↓ /api/v1/privada/*
Cloud Run svc-privada (proyecto essential-haiku-482815-u4)
    ↓
BigQuery infra_gestion
```

**Cambio requerido en el sistema existente**: agregar el dominio del nuevo API Gateway a la lista `allow_origins` en `backend/app/main.py`.

**Compatibilidad de auth**: ambos sistemas usan Google OAuth 2.0 con issuer `accounts.google.com`. El nuevo API Gateway está configurado con el mismo issuer, por lo que los tokens son compatibles — verificar con prueba end-to-end.

---

## Ubicación del código fuente

```
C:\Users\pbonafe\OneDrive - Getronics\Documents\programacion\proyecto_sistema_gestiones\
├── Sistema-Gestiones-Internas-Infraestructura-main\   (backend FastAPI)
├── front_infraestructura V2\                           (frontend Vanilla JS)
├── informe\                                            (informes HTML + scripts BQ)
└── contexto\                                           (documentación del sistema)
```
