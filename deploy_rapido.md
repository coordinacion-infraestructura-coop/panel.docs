#Verificar q el proxy este activo o levantarlo
~/cloud-sql-proxy gestorcooperativo:southamerica-east1:ministerio-postgres --port 5432 &

# Marcar pass
export DATABASE_URL=$(python3 -c "
import urllib.parse
pw = 'PUgSJkQQMyFCl2BkJDThXwwG01w+CjSy22VRjFNWcq0='
print(f'postgresql+asyncpg://user_vivienda:{urllib.parse.quote(pw, safe=\"\")}@127.0.0.1:5433/db_vivienda')
")

# Verificar
python -m alembic current

correr
cd services/
git pull
cd svc-vivienda # o el modulo que estemos editando

python -m alembic current
python -m alembic upgrade head

# Run deploy backend
## Local 
git add , commit y push origin_back

## Cloud Shell

cd ~/gestorcooperativo/backend/svc-vivienda
git pull origin main
gcloud run deploy svc-vivienda \
  --source . \
  --region=southamerica-east1 \
  --project=gestorcooperativo

# Run deploy gateway
## Local
cd infra
git add -A , commit y push origin_infra

## Cloud shell

cd ~/panel.infra/gateway
git pull origin main

# Config nueva (nombre único — fecha de hoy)
gcloud api-gateway api-configs create ministerio-config-v20260704b \
  --api=ministerio-api \
  --openapi-spec=openapi.yaml \
  --project=gestorcooperativo \
  --backend-auth-service-account=api-gateway-sa@gestorcooperativo.iam.gserviceaccount.com

# Apuntar el gateway a la nueva config
gcloud api-gateway gateways update ministerio-gateway \
  --api=ministerio-api \
  --api-config=ministerio-config-v20260704b \
  --location=us-central1 \
  --project=gestorcooperativo


# Run deploy frontend
firebase deploy --only hosting --project gestorcooperativo

# Orden crítico
migraciones alembick → deploy backend cloud Run (para que los endpoints nuevos existan)  → Gateway (para que los ruteé) → Frontend (para que los llame). Si deployás el frontend antes del Gateway, las llamadas a los endpoints nuevos van a devolver 404.