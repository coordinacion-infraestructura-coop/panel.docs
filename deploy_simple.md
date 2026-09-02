# Deploy rápido — GestorCooperativo

Referencia compacta. Para detalles y errores conocidos ver `docs/deploy.md`.

---

## Cuándo ejecutar cada paso

| Cambio | Pasos necesarios |
|--------|-----------------|
| Solo lógica Python (bug fix, ajuste) | 1 |
| Nueva tabla o columna | 0 → 1 |
| Nuevo endpoint en FastAPI | 1 → 2 |
| Nueva tabla + nuevo endpoint + UI | 0 → 1 → 2 → 3 |
| Solo UI / estilos | 3 |
| Solo CORS u opciones del gateway | 2 |

---

## Paso 0 — Migración Alembic (Cloud Shell)

```bash
# Iniciar Cloud SQL Auth Proxy en puerto 5433 (evita conflicto con PG local)
~/cloud-sql-proxy gestorcooperativo:southamerica-east1:ministerio-postgres --port 5433 &
sleep 4

# Setear DATABASE_URL con password URL-encodeado
export DATABASE_URL=$(python3 -c "
import urllib.parse
pw = 'PUgSJkQQMyFCl2BkJDThXwwG01w+CjSy22VRjFNWcq0='
print(f'postgresql+asyncpg://user_vivienda:{urllib.parse.quote(pw, safe=\"\")}@127.0.0.1:5433/db_vivienda')
")

# Pararse en el directorio correcto (alembic.ini usa rutas relativas)
cd ~/gestorcooperativo/backend/svc-vivienda
git pull origin main

python -m alembic upgrade head
python -m alembic current   # confirmar que muestra la revisión esperada (head)
```

---

## Paso 1 — Cloud Run (Cloud Shell)

```bash
cd ~/gestorcooperativo/backend/svc-vivienda
git pull origin main   # SIEMPRE antes de deployar

gcloud run deploy svc-vivienda \
  --source . \
  --region=southamerica-east1 \
  --project=gestorcooperativo

# Verificar (Cloud Run requiere auth — usar identity token del SA activo)
curl -s -H "Authorization: Bearer $(gcloud auth print-identity-token)" \
  https://svc-vivienda-iwni7vc2qq-rj.a.run.app/health
# Esperado: {"status":"ok","service":"svc-vivienda","version":"0.1.0"}
# Un 403 sin el header es NORMAL — no indica error de deploy
```

---

## Paso 2 — API Gateway (Cloud Shell)

> Las configs del gateway son **inmutables**. Cada cambio necesita un nombre nuevo.
> Formato: `ministerio-config-v{YYYYMMDD}` o `ministerio-config-v{YYYYMMDD}b`, `c`, etc.

```bash
cd ~/gestorcooperativo/infra/gateway
git pull origin main

# Verificar que el YAML tiene las rutas con OPTIONS (debe ser >= 15)
grep -c 'operationId: cors' openapi.yaml

# Crear nueva config (cambiar el sufijo si ya existe ese nombre)
gcloud api-gateway api-configs create ministerio-config-v20260827 \
  --api=ministerio-api \
  --openapi-spec=openapi.yaml \
  --project=gestorcooperativo \
  --backend-auth-service-account=api-gateway-sa@gestorcooperativo.iam.gserviceaccount.com

# Apuntar el gateway a la nueva config
gcloud api-gateway gateways update ministerio-gateway \
  --api=ministerio-api \
  --api-config=ministerio-config-v20260827 \
  --location=us-central1 \
  --project=gestorcooperativo

# Esperar ~5 min y verificar CORS en algún endpoint clave
curl -s -o /dev/null -w "HTTP %{http_code}\n" -X OPTIONS \
  "https://ministerio-gateway-3j5k00ma.uc.gateway.dev/api/v1/vivienda/cordoba-hogar" \
  -H "Origin: https://gestorcooperativo.web.app"
# Esperado: HTTP 200
```

---

## Paso 3 — Frontend (Windows local — NUNCA desde Cloud Shell)

```powershell
cd "C:\Users\pbonafe\OneDrive - Getronics\Documents\programacion\GestorCooperativo\frontend"
npm run build
# Debe terminar con "✓ built in X.XXs" — si hay errores TypeScript el deploy no sirve

firebase deploy --only hosting --project gestorcooperativo
# URL: https://gestorcooperativo.web.app
```

---

## Historial de configs del gateway

| Config | Fecha | Estado |
|--------|-------|--------|
| ministerio-config-v20260701 | 2026-07-01 | ❌ sin OPTIONS |
| ministerio-config-v20260701-cors | 2026-07-01 | ✅ reemplazada |
| ministerio-config-v20260701b | 2026-07-01 | ✅ reemplazada |
| ministerio-config-v20260701c | 2026-07-01 | ✅ reemplazada |
| ministerio-config-v20260701d | 2026-07-01 | ✅ reemplazada |
| ministerio-config-v20260702a | 2026-07-02 | ✅ reemplazada |
| ministerio-config-v20260702b | 2026-07-02 | ✅ reemplazada |
| ministerio-config-v20260703a | 2026-07-03 | ✅ reemplazada |
| ministerio-config-v20260704a | 2026-07-04 | ⚠ posible YAML viejo |
| ministerio-config-v20260704b | 2026-07-04 | ✅ reemplazada |
| ministerio-config-v20260713 | 2026-07-13 | ✅ **ACTIVA** — agrega GET checklist-tecnico Cordón Cuneta |

---

## Historial de migraciones

| Revisión | Fecha | Descripción |
|----------|-------|-------------|
| 0001 | 2026-06-29 | Schema inicial (CC, beneficiarios, expedientes, audit) |
| 0002 | 2026-07-01 | Seed: 4 programas + 46 municipios CC |
| 0003 | 2026-07-01 | viv_cc_pedidos |
| 0004 | 2026-07-02 | portal_usuarios + seed admins |
| 0005 | 2026-07-02 | Sync datos CC desde Panel_Cordon_Cuneta(10).html |
| 0006 | 2026-07-02 | Córdoba Hogar: tablas + seed 43 localidades |
| 0007 | 2026-07-04 | Schema: estado_general, deleted_at, historial CC/CH, geo_localidades, aplica_* flags |
| 0008 | 2026-07-04 | Data: sync CC desde Panel(15).html, seed geo_localidades, computa estado_general |
| 0009 | 2026-07-04 | Unifica catálogos CC y CH al workflow de 15 estados estándar |
| 0010 | 2026-07-04 | Migra obs → pedidos historial, doc_exp fechado → estado historial |
| 0011 | 2026-07-05 | Actualiza CC desde Panel #25: estados + métricas; inserta 8 nuevos municipios |
| 0012 | 2026-07-05 | Backfill historial CC faltante de Panel #25 (estados sin cambio entre Panel 15 y 25) |
| 0013 | 2026-07-08 | `monto_por_casa` configurable en `viv_ch_config` |
| 0014 | 2026-07-13 | Índice único `(municipio, departamento)` activo en `viv_cordon_cuneta` |
| 0015 | 2026-07-13 | Índice único `(localidad, departamento)` activo en `viv_cordoba_hogar` |
| 0016 | 2026-07-13 | `viv_cc_checklist_tecnico` + `viv_cc_checklist_items` + `viv_cc_sync_log` — sync Google Sheet "Base TOTAL" |

---

## Notas críticas

- **`git pull` SIEMPRE antes de deployar** — sin pull, Cloud Run sube código viejo
- **Frontend SOLO desde Windows** — Cloud Shell puede tener código desactualizado
- **`cd svc-vivienda/` para migrations** — alembic.ini usa rutas relativas
- **Password Cloud SQL**: `PUgSJkQQMyFCl2BkJDThXwwG01w+CjSy22VRjFNWcq0=` → URL-encodear con `urllib.parse.quote`
- **Puerto 5433** para el proxy (evita conflicto con PG local que ocupa 5432)
- **Errores CORS** después de deploy → verificar Health de Cloud Run primero, luego el preflight OPTIONS del gateway
- **`gcloud run deploy --source .` corrido desde el directorio padre** (ej. `backend/` en vez de `backend/svc-vivienda/`) no encuentra el `Dockerfile` y cae a Buildpacks — el contenedor resultante falla el health check de arranque. Siempre `cd` al directorio del servicio antes de deployar; si un deploy fallido dejó un base-image residual, agregar `--clear-base-image` al reintentar.
- **`curl` sin `Authorization` a un endpoint de Cloud Run privado devuelve 403 — es normal**, no indica error de deploy. Usar `-H "Authorization: Bearer $(gcloud auth print-identity-token)"` para probar.
- **Endpoints internos disparados por Cloud Scheduler** (ej. `/internal/sync/...`, ver `spec-sync-cc-checklist-tecnico.md`): no se agregan a `openapi.yaml` ni pasan por el Gateway. Se protegen únicamente con `roles/run.invoker` de Cloud Run sobre una Service Account dedicada, y Cloud Scheduler llama directo a la URL de Cloud Run con token OIDC (`--oidc-service-account-email` + `--oidc-token-audience`). Patrón reutilizable para cualquier sync/batch futuro disparado por Scheduler.
