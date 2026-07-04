# Guía de Deploy — GestorCooperativo

Pasos para redesplegar cada capa del sistema. Ejecutar siempre en el orden indicado cuando haya cambios en múltiples capas.

---

## 1. Backend (Cloud Run — svc-vivienda)

Desde Cloud Shell o terminal con `gcloud` configurado:

```bash
cd ~/gestorcooperativo/backend/svc-vivienda   # <-- IMPORTANTE: pararse aquí exactamente
git pull origin main                           # <-- SIEMPRE antes de deployar
gcloud run deploy svc-vivienda \
  --source . \
  --region=southamerica-east1 \
  --project=gestorcooperativo
```

> **`--source .` sube el directorio actual como contexto del build.** Si estás en
> `infra/gateway/` o en la raíz del repo, el buildpack no encuentra los archivos Python
> y falla con `No buildpack groups passed detection`.
>
> **Sin `git pull` el deploy sube código viejo.** Cloud Run queda corriendo la versión
> anterior y los endpoints nuevos devuelven 404 aunque el gateway los reconozca.

El deploy tarda ~3 minutos. Cloud Run hace rolling update sin downtime.

---

## 2. API Gateway

Necesario **solo cuando cambia `infra/gateway/openapi.yaml`** (nuevos endpoints, CORS, rutas).

```bash
cd ~/panel.infra/gateway
git pull origin main

# Crear nueva versión del config (el nombre debe ser único — usar fecha+sufijo)
gcloud api-gateway api-configs create ministerio-config-v{YYYYMMDD} \
  --api=ministerio-api \
  --openapi-spec=openapi.yaml \
  --project=gestorcooperativo \
  --backend-auth-service-account=api-gateway-sa@gestorcooperativo.iam.gserviceaccount.com

# Apuntar el gateway a la nueva config
gcloud api-gateway gateways update ministerio-gateway \
  --api=ministerio-api \
  --api-config=ministerio-config-v{YYYYMMDD} \
  --location=us-central1 \
  --project=gestorcooperativo
```

> Si `api-configs create` falla con `ALREADY_EXISTS`, el config con ese nombre ya existe
> (posiblemente creado con el spec viejo). Usar un sufijo diferente: `ministerio-config-v{YYYYMMDD}b`.

El gateway tarda ~5 minutos en propagar. Verificar con:
```bash
curl -s https://ministerio-gateway-3j5k00ma.uc.gateway.dev/health
```

---

## 3. Frontend (Firebase Hosting)

> **CRÍTICO: Ejecutar SIEMPRE desde la máquina Windows local, NUNCA desde Cloud Shell.**
> 
> Cloud Shell puede tener una copia desactualizada del código. Si `tsc -b` falla (por errores TypeScript), Vite no corre y la carpeta `dist/` queda con el build anterior — Firebase despliega el código viejo silenciosamente sin avisar.

```powershell
# Windows — desde el directorio frontend del proyecto
cd "C:\Users\pbonafe\OneDrive - Getronics\Documents\programacion\GestorCooperativo\frontend"
npm run build
# Verificar: debe mostrar "✓ built in X.XXs" — si hay errores TypeScript el build no corre

firebase deploy --only hosting --project gestorcooperativo
```

La URL del sitio: https://gestorcooperativo.web.app

> Si `firebase deploy` falla con "Failed to authenticate", ejecutar `firebase login` primero.
> 
> El script `npm run build` ejecuta `tsc -b && vite build`. Si TypeScript reporta errores (`noUnusedLocals`, `noUnusedParameters`, tipos faltantes), el build se detiene en `tsc -b` y la carpeta `dist/` mantiene el contenido anterior. Siempre confirmar visualmente que el output termina con `✓ built in`.

---

## 4. Backend svc-privada (Cloud Run — proyecto essential-haiku-482815-u4)

El sistema de gestiones existente se deploya en un **proyecto GCP separado**:

```bash
# Desde Cloud Shell asegurarse de estar en el proyecto correcto
gcloud config set project essential-haiku-482815-u4

# El código está en el repo del sistema de gestiones (no en panel.backend)
cd ~/Sistema-Gestiones-Internas-Infraestructura-main/backend/app   # <-- ruta exacta en Cloud Shell
git pull origin main   # si el repo tiene remote configurado; si no, copiar los archivos manualmente

gcloud run deploy infraestructura-gestioninterna \
  --source . \
  --region=southamerica-east1 \
  --project=essential-haiku-482815-u4
```

**Primera vez — IAM cross-project (ejecutar una sola vez — ✅ EJECUTADO 2026-07-02):**

Para que el API Gateway del proyecto `gestorcooperativo` pueda invocar el Cloud Run de `essential-haiku-482815-u4`:

```bash
gcloud run services add-iam-policy-binding infraestructura-gestioninterna \
  --member="serviceAccount:api-gateway-sa@gestorcooperativo.iam.gserviceaccount.com" \
  --role="roles/run.invoker" \
  --region=southamerica-east1 \
  --project=essential-haiku-482815-u4
```

> Sin este binding el gateway obtiene 403 al intentar llamar al svc-privada, aunque el JWT del usuario sea válido.

---

## 5. Migraciones Alembic (Cloud SQL)

Necesario cuando se agregan nuevas tablas o columnas. Requiere Cloud SQL Proxy para conectar desde Cloud Shell.

```bash
# 1. Descargar Cloud SQL Proxy si no está disponible
curl -o ~/cloud-sql-proxy \
  https://storage.googleapis.com/cloud-sql-connectors/cloud-sql-proxy/v2.14.1/cloud-sql-proxy.linux.amd64
chmod +x ~/cloud-sql-proxy

# 2. Levantar el proxy en background
~/cloud-sql-proxy gestorcooperativo:southamerica-east1:ministerio-postgres \
  --port 5432 > /tmp/proxy.log 2>&1 &

# Esperar y verificar
sleep 4 && cat /tmp/proxy.log
# Debe mostrar: [gestorcooperativo:southamerica-east1:ministerio-postgres] Listening on 127.0.0.1:5432

# 3. Obtener el password del secret (si no lo tenés a mano)
gcloud secrets versions access latest --secret="svc-vivienda-db-url" --project=gestorcooperativo

# 4. Pararse en el directorio correcto del servicio
cd ~/gestorcooperativo/backend/svc-vivienda   # <-- CRÍTICO: alembic.ini usa rutas relativas

# 5. Correr la migración con TCP URL (no socket URL)
# Importante: + → %2B, = → %3D en el password si tiene esos caracteres
DATABASE_URL="postgresql+asyncpg://user_vivienda:TU_PASSWORD_ENCODEADO@127.0.0.1:5432/db_vivienda" \
  python -m alembic upgrade head

#pass completa y encodeada
DATABASE_URL="postgresql+asyncpg://user_vivienda:PUgSJkQQMyFCl2BkJDThXwwG01w%2BCjSy22VRjFNWcq0%3D@127.0.0.1:5432/db_vivienda"

# 6. Verificar
python -m alembic current
```

> **Errores comunes:**
> - `No 'script_location' key found` → estás en el directorio equivocado. Ir a `svc-vivienda/`.
> - `[Errno 111] Connect call failed` → el proxy no está corriendo o cayó. Verificar con `jobs` y relanzar.
> - `Connect call failed ('127.0.0.1', 5432)` → estás usando socket URL. Cambiar `?host=/cloudsql/...` por `@127.0.0.1:5432`.
> - Error en password → URL-encodear los caracteres especiales: `+` → `%2B`, `=` → `%3D`.

**Historial de migraciones ejecutadas en producción:**

| Revisión | Fecha | Descripción |
|----------|-------|-------------|
| 0001 | 2026-06-29 | Schema inicial (viv_programas, viv_beneficiarios, viv_expedientes, viv_historial_expedientes, viv_asignaciones, viv_cordon_cuneta, audit_log) |
| 0002 | 2026-07-01 | Seed data (4 programas, 46 municipios Cordón Cuneta) |
| 0003 | 2026-07-01 | viv_cc_pedidos (pedidos Cordón Cuneta) |
| 0004 | 2026-07-02 | portal_usuarios + portal_usuario_secretarias + seed admins |
| 0005 | 2026-07-02 | update_cc_data — sincroniza datos CC con Panel_Cordon_Cuneta(10).html |
| 0006 | 2026-07-02 | cordoba_hogar — viv_ch_estados, viv_cordoba_hogar, viv_ch_config, viv_ch_pedidos |

---

## 6. Seed scripts

Algunos módulos tienen scripts de seed independientes que deben correrse después de las migraciones.

### Córdoba Hogar (`seed_cordoba_hogar.py`)

Requiere: migración 0006 aplicada, Cloud SQL Proxy activo.

```bash
# 1. Levantar el proxy (si no está)
~/cloud-sql-proxy gestorcooperativo:southamerica-east1:ministerio-postgres --port 5432 &

# 2. Obtener password del secret
gcloud secrets versions access latest --secret="svc-vivienda-db-url" --project=gestorcooperativo
# Devuelve: postgresql+asyncpg://user_vivienda:PASSWORD@/db_vivienda?host=/cloudsql/...
# Extraer PASSWORD y URL-encodear: + → %2B, = → %3D

# 3. Pararse en el directorio del servicio
cd ~/gestorcooperativo/backend/svc-vivienda

# 4. Correr el seed (con TCP URL, no socket URL)
DATABASE_URL="postgresql+asyncpg://user_vivienda:PASSWORD_ENCODEADO@127.0.0.1:5432/db_vivienda" \
  python seed_cordoba_hogar.py
```

**Errores conocidos del seed:**

| Error | Causa | Fix |
|-------|-------|-----|
| `password authentication failed for user "svc_vivienda"` | Usuario incorrecto | Usar `user_vivienda` (no `svc_vivienda`) |
| `password authentication failed for user "user_vivienda"` | Password con `+`/`=` sin encodear | URL-encodear: `+` → `%2B`, `=` → `%3D` |
| `'str' object has no attribute 'toordinal'` | Campo DATE recibe string ISO | El fix está en seed_cordoba_hogar.py: `date.fromisoformat()` convierte antes de insertar |

---

## Orden recomendado cuando cambian varias capas

| Cambio | Qué redesplegar |
|--------|----------------|
| Solo lógica de negocio / bug en Python | Backend |
| Nuevo endpoint en FastAPI | Backend → API Gateway |
| Nueva tabla en BD | Migración Alembic → Backend |
| Nueva tabla + nuevo endpoint + UI | Migración → Backend → API Gateway → Frontend |
| Solo UI / estilos | Frontend |
| Nuevo endpoint + UI que lo consume | Backend → API Gateway → Frontend |
| Solo CORS u otras config del gateway | API Gateway |

---

## Versiones de config del gateway desplegadas

| Config | Fecha | Descripción | Estado |
|--------|-------|-------------|--------|
| ministerio-config-v20260701 | 2026-07-01 | Creada con spec viejo (sin OPTIONS) | ❌ no usar |
| ministerio-config-v20260701-cors | 2026-07-01 | Primera versión con OPTIONS en todos los paths | ✅ reemplazada |
| ministerio-config-v20260701b | 2026-07-01 | Agrega DELETE /pedidos/{pedido_id} | ✅ reemplazada |
| ministerio-config-v20260701c | 2026-07-01 | OPTIONS + DELETE pedidos | ✅ reemplazada |
| ministerio-config-v20260701d | 2026-07-01 | APPEND_PATH_TO_ADDRESS en paths svc-privada | ✅ reemplazada |
| ministerio-config-v20260702a | 2026-07-02 | Rutas portal (/api/v1/portal/*) | ✅ reemplazada |
| ministerio-config-v20260702b | 2026-07-02 | Rutas portal + corrección CORS | ✅ reemplazada |
| ministerio-config-v20260703a | 2026-07-02 | **✅ ACTIVA** — Rutas Córdoba Hogar (`/api/v1/vivienda/cordoba-hogar*`) + OPTIONS | ✅ ACTIVA |
