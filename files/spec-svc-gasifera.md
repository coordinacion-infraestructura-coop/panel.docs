# Spec: svc-gasifera — Secretaría de Infraestructura Gasífera

**Estado**: draft  
**Versión**: 0.1.0  
**Última actualización**: 2025-01

---

## 1. Propósito

Servicio backend para la Secretaría de Infraestructura Gasífera. Gestiona tres áreas diferenciadas: el Programa de Conexión de Gas en Escuelas, los Asesoramientos Legales y Contables, y los Créditos para Desarrollo de Infraestructura ejecutados por cooperativas.

## 2. Módulos internos

| Módulo | Descripción |
|--------|-------------|
| `escuelas` | Programa de conexión de gas en escuelas |
| `asesoramientos` | Turnos y seguimiento de asesoramientos legales/contables |
| `creditos` | Créditos para cooperativas, amortización y seguimiento |

---

## 3. Modelos de datos

### Módulo `escuelas`

#### `gas_escuelas`

| Columna | Tipo | Descripción |
|---------|------|-------------|
| `id` | UUID | PK |
| `cue` | VARCHAR(20) | Código Único de Establecimiento |
| `nombre` | VARCHAR(200) | Nombre de la escuela |
| `localidad` | VARCHAR(100) | |
| `departamento` | VARCHAR(100) | |
| `domicilio` | VARCHAR(300) | |
| `nivel` | ENUM | `INICIAL/PRIMARIO/SECUNDARIO/TECNICO` |
| `estado_conexion` | ENUM | ver abajo |
| `fecha_relevamiento` | DATE | |
| `fecha_inicio_obra` | DATE | NULL |
| `fecha_fin_obra` | DATE | NULL |
| `cooperativa_ejecutora_id` | UUID | NULL — FK → gas_cooperativas_ejecutoras |
| `observaciones` | TEXT | |
| `created_at` | TIMESTAMPTZ | |
| `updated_at` | TIMESTAMPTZ | |
| `deleted_at` | TIMESTAMPTZ | NULL |

Estados conexión escuela:
```python
class EstadoConexion(str, Enum):
    RELEVADA = "RELEVADA"
    EN_PROYECTO = "EN_PROYECTO"
    LICITADA = "LICITADA"
    EN_EJECUCION = "EN_EJECUCION"
    FINALIZADA = "FINALIZADA"
    HABILITADA = "HABILITADA"
```

### Módulo `asesoramientos`

#### `gas_asesoramientos`

| Columna | Tipo | Descripción |
|---------|------|-------------|
| `id` | UUID | PK |
| `tipo` | ENUM | `LEGAL / CONTABLE` |
| `solicitante_razon_social` | VARCHAR(200) | Cooperativa o mutual solicitante |
| `solicitante_cuit` | VARCHAR(13) | |
| `fecha_turno` | TIMESTAMPTZ | |
| `profesional_asignado` | VARCHAR(200) | |
| `tema` | TEXT | Descripción del tema a asesorar |
| `estado` | ENUM | `PENDIENTE/CONFIRMADO/REALIZADO/CANCELADO` |
| `resolucion` | TEXT | NULL — nota del profesional post-asesoramiento |
| `created_at` | TIMESTAMPTZ | |
| `updated_at` | TIMESTAMPTZ | |
| `deleted_at` | TIMESTAMPTZ | NULL |
| `created_by` | VARCHAR(100) | |

### Módulo `creditos`

#### `gas_creditos`

| Columna | Tipo | Descripción |
|---------|------|-------------|
| `id` | UUID | PK |
| `numero_credito` | VARCHAR(30) | GAS-YYYY-NNNNNN |
| `cooperativa_id` | UUID | FK → gas_cooperativas_ejecutoras |
| `monto_aprobado` | NUMERIC(14,2) | |
| `moneda` | VARCHAR(3) | DEFAULT 'ARS' |
| `tasa_interes` | NUMERIC(5,2) | % anual |
| `plazo_meses` | INTEGER | |
| `cuota_mensual` | NUMERIC(12,2) | Calculada al aprobar |
| `estado` | ENUM | ver abajo |
| `fecha_aprobacion` | DATE | NULL |
| `fecha_primer_vencimiento` | DATE | NULL |
| `descripcion_proyecto` | TEXT | |
| `created_at` | TIMESTAMPTZ | |
| `updated_at` | TIMESTAMPTZ | |
| `deleted_at` | TIMESTAMPTZ | NULL |
| `created_by` | VARCHAR(100) | |

Estados crédito:
```python
class EstadoCredito(str, Enum):
    SOLICITADO = "SOLICITADO"
    EN_EVALUACION = "EN_EVALUACION"
    APROBADO = "APROBADO"
    DESEMBOLSADO = "DESEMBOLSADO"
    EN_MORA = "EN_MORA"
    CANCELADO = "CANCELADO"
    INCOBRABLE = "INCOBRABLE"
```

#### `gas_cuotas`

| Columna | Tipo | Descripción |
|---------|------|-------------|
| `id` | UUID | PK |
| `credito_id` | UUID | FK → gas_creditos |
| `numero_cuota` | INTEGER | 1..N |
| `fecha_vencimiento` | DATE | |
| `monto` | NUMERIC(12,2) | |
| `monto_pagado` | NUMERIC(12,2) | DEFAULT 0 |
| `fecha_pago` | DATE | NULL |
| `estado` | ENUM | `PENDIENTE/PAGADA/VENCIDA/PARCIAL` |
| `recargo` | NUMERIC(12,2) | DEFAULT 0 |

#### `gas_cooperativas_ejecutoras`

| Columna | Tipo | Descripción |
|---------|------|-------------|
| `id` | UUID | PK |
| `cuit` | VARCHAR(13) | UNIQUE |
| `razon_social` | VARCHAR(200) | |
| `matricula` | VARCHAR(50) | |
| `localidad` | VARCHAR(100) | |
| `departamento` | VARCHAR(100) | |
| `email_contacto` | VARCHAR(200) | |
| `telefono` | VARCHAR(20) | |
| `activa` | BOOLEAN | DEFAULT true |
| `created_at` | TIMESTAMPTZ | |

---

## 4. Endpoints

### Escuelas

```
GET    /api/v1/gasifera/escuelas
POST   /api/v1/gasifera/escuelas
GET    /api/v1/gasifera/escuelas/{id}
PATCH  /api/v1/gasifera/escuelas/{id}
POST   /api/v1/gasifera/escuelas/{id}/transicion
GET    /api/v1/gasifera/escuelas/por-departamento/{departamento}
GET    /api/v1/gasifera/escuelas/estadisticas
```

### Asesoramientos

```
GET    /api/v1/gasifera/asesoramientos
POST   /api/v1/gasifera/asesoramientos
GET    /api/v1/gasifera/asesoramientos/{id}
PATCH  /api/v1/gasifera/asesoramientos/{id}
POST   /api/v1/gasifera/asesoramientos/{id}/resolver
GET    /api/v1/gasifera/asesoramientos/disponibilidad?fecha={date}&tipo={tipo}
```

### Créditos

```
GET    /api/v1/gasifera/creditos
POST   /api/v1/gasifera/creditos
GET    /api/v1/gasifera/creditos/{id}
PATCH  /api/v1/gasifera/creditos/{id}
POST   /api/v1/gasifera/creditos/{id}/transicion
GET    /api/v1/gasifera/creditos/{id}/cuotas
POST   /api/v1/gasifera/creditos/{id}/cuotas/{cuota_id}/pagar
GET    /api/v1/gasifera/creditos/en-mora
```

### Cooperativas ejecutoras

```
GET    /api/v1/gasifera/cooperativas
POST   /api/v1/gasifera/cooperativas
GET    /api/v1/gasifera/cooperativas/{id}
PATCH  /api/v1/gasifera/cooperativas/{id}
GET    /api/v1/gasifera/cooperativas/{id}/creditos
GET    /api/v1/gasifera/cooperativas/{id}/escuelas
```

---

## 5. Reglas de negocio

### Créditos
1. Al aprobar un crédito, se generan automáticamente todas las cuotas del plan de amortización (sistema francés).
2. Una cooperativa no puede tener más de 2 créditos activos simultáneamente (estado `DESEMBOLSADO`).
3. Una cuota pasa a `VENCIDA` automáticamente si `fecha_vencimiento < hoy` y `estado == PENDIENTE`.
4. Si una cooperativa tiene 3 cuotas `VENCIDAS` consecutivas, el crédito pasa a `EN_MORA`.
5. El recargo por mora se calcula como 2% mensual sobre el saldo pendiente.

### Asesoramientos
1. No se pueden agendar dos asesoramientos para el mismo profesional en el mismo bloque horario.
2. Un asesoramiento `REALIZADO` debe tener `resolucion` no nula.

---

## 6. Eventos Pub/Sub publicados

| Event type | Disparador |
|-----------|------------|
| `gasifera.credito.aprobado` | Transición → APROBADO |
| `gasifera.credito.desembolsado` | Transición → DESEMBOLSADO |
| `gasifera.credito.en_mora` | Transición automática → EN_MORA |
| `gasifera.cuota.pagada` | POST pagar cuota |
| `gasifera.escuela.habilitada` | Transición → HABILITADA |
| `gasifera.asesoramiento.confirmado` | PATCH estado → CONFIRMADO |

---

## 7. Criterios de aceptación

- [ ] CRUD completo para los tres módulos
- [ ] Generación automática de plan de cuotas al aprobar crédito (sistema francés)
- [ ] Job/tarea programada para marcar cuotas vencidas (Cloud Scheduler → endpoint interno)
- [ ] Transición automática a EN_MORA con 3 cuotas vencidas consecutivas
- [ ] Cálculo de recargo por mora
- [ ] Disponibilidad de agenda para asesoramientos
- [ ] Tests unitarios de la lógica de amortización
- [ ] Cobertura > 80%
