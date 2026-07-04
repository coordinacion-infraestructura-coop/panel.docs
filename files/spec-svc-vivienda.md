# Spec: svc-vivienda — Secretaría de Vivienda

**Estado**: draft  
**Versión**: 0.1.0  
**Responsable de spec**: pendiente asignar  
**Última actualización**: 2025-01

---

## 1. Propósito

Servicio backend para la Secretaría de Vivienda. Gestiona los programas habitacionales provinciales: Córdoba Hogar, Mi Lugar, Cordón Cuneta y Programa de Loteos. Permite registrar beneficiarios, hacer seguimiento de expedientes y reportar estado de avance a la Dirección de Vivienda.

## 2. Alcance

### Incluido en este servicio
- Gestión de beneficiarios (alta, baja, modificación, consulta)
- Gestión de expedientes por programa
- Seguimiento de estados de expediente
- Asignación de viviendas / lotes
- Historial de auditoría por expediente
- Reportes de avance por programa

### Fuera de alcance (otros servicios)
- Autenticación y roles → `svc-auth`
- Notificaciones a beneficiarios → `svc-notificaciones`
- Documentos adjuntos → `svc-documentos`
- Dashboard ejecutivo → BigQuery / frontend módulo dashboard

---

## 3. Módulos internos

| Módulo | Descripción |
|--------|-------------|
| `beneficiarios` | Registro de personas beneficiarias |
| `expedientes` | Ciclo de vida de solicitudes/expedientes |
| `programas` | Catálogo de programas y sus configuraciones |
| `asignaciones` | Asignación vivienda/lote a beneficiario |

---

## 4. Modelos de datos

### 4.1 `viv_beneficiarios`

| Columna | Tipo | Restricciones | Descripción |
|---------|------|---------------|-------------|
| `id` | UUID | PK | |
| `dni` | VARCHAR(10) | NOT NULL, UNIQUE | Documento nacional |
| `cuil` | VARCHAR(13) | NOT NULL | |
| `nombre` | VARCHAR(100) | NOT NULL | |
| `apellido` | VARCHAR(100) | NOT NULL | |
| `fecha_nacimiento` | DATE | NOT NULL | |
| `email` | VARCHAR(200) | | |
| `telefono` | VARCHAR(20) | | |
| `domicilio_calle` | VARCHAR(200) | | |
| `domicilio_numero` | VARCHAR(10) | | |
| `domicilio_localidad` | VARCHAR(100) | | |
| `domicilio_departamento` | VARCHAR(100) | | |
| `grupo_familiar_count` | INTEGER | DEFAULT 1 | |
| `ingreso_mensual` | NUMERIC(12,2) | | |
| `created_at` | TIMESTAMPTZ | NOT NULL DEFAULT now() | |
| `updated_at` | TIMESTAMPTZ | NOT NULL DEFAULT now() | |
| `deleted_at` | TIMESTAMPTZ | NULL | Soft delete |
| `created_by` | VARCHAR(100) | NOT NULL | UID Firebase |

### 4.2 `viv_programas`

| Columna | Tipo | Restricciones | Descripción |
|---------|------|---------------|-------------|
| `id` | UUID | PK | |
| `codigo` | VARCHAR(50) | NOT NULL, UNIQUE | Ej: `CORDOBA_HOGAR` |
| `nombre` | VARCHAR(200) | NOT NULL | |
| `descripcion` | TEXT | | |
| `activo` | BOOLEAN | DEFAULT true | |
| `requiere_ingreso_max` | BOOLEAN | DEFAULT false | |
| `ingreso_max` | NUMERIC(12,2) | NULL | |
| `created_at` | TIMESTAMPTZ | NOT NULL DEFAULT now() | |
| `updated_at` | TIMESTAMPTZ | NOT NULL DEFAULT now() | |

Seed inicial:
```sql
INSERT INTO viv_programas (id, codigo, nombre) VALUES
  (gen_random_uuid(), 'CORDOBA_HOGAR', 'Córdoba Hogar'),
  (gen_random_uuid(), 'MI_LUGAR', 'Mi Lugar'),
  (gen_random_uuid(), 'CORDON_CUNETA', 'Cordón Cuneta'),
  (gen_random_uuid(), 'LOTEOS', 'Programa de Loteos');
```

### 4.3 `viv_expedientes`

| Columna | Tipo | Restricciones | Descripción |
|---------|------|---------------|-------------|
| `id` | UUID | PK | |
| `numero_expediente` | VARCHAR(30) | NOT NULL, UNIQUE | Generado: VIV-YYYY-NNNNNN |
| `beneficiario_id` | UUID | FK → viv_beneficiarios | |
| `programa_id` | UUID | FK → viv_programas | |
| `estado` | ENUM | NOT NULL | ver estados abajo |
| `fecha_solicitud` | DATE | NOT NULL | |
| `fecha_resolucion` | DATE | NULL | |
| `observaciones` | TEXT | | |
| `prioridad` | SMALLINT | DEFAULT 3 | 1=alta, 2=media, 3=normal |
| `created_at` | TIMESTAMPTZ | NOT NULL DEFAULT now() | |
| `updated_at` | TIMESTAMPTZ | NOT NULL DEFAULT now() | |
| `deleted_at` | TIMESTAMPTZ | NULL | |
| `created_by` | VARCHAR(100) | NOT NULL | |
| `updated_by` | VARCHAR(100) | NOT NULL | |

Estados del expediente:
```python
class EstadoExpediente(str, Enum):
    INGRESADO = "INGRESADO"
    EN_EVALUACION = "EN_EVALUACION"
    DOCUMENTACION_PENDIENTE = "DOCUMENTACION_PENDIENTE"
    APROBADO = "APROBADO"
    EN_LISTA_ESPERA = "EN_LISTA_ESPERA"
    ASIGNADO = "ASIGNADO"
    RECHAZADO = "RECHAZADO"
    BAJA = "BAJA"
```

Transiciones válidas:
```
INGRESADO → EN_EVALUACION
EN_EVALUACION → DOCUMENTACION_PENDIENTE | APROBADO | RECHAZADO
DOCUMENTACION_PENDIENTE → EN_EVALUACION
APROBADO → EN_LISTA_ESPERA
EN_LISTA_ESPERA → ASIGNADO
ASIGNADO → BAJA (por fallecimiento u otras causas excepcionales)
RECHAZADO → INGRESADO (re-ingreso con nueva documentación)
```

### 4.4 `viv_historial_expedientes`

| Columna | Tipo | Descripción |
|---------|------|-------------|
| `id` | UUID | PK |
| `expediente_id` | UUID | FK |
| `estado_anterior` | VARCHAR(50) | |
| `estado_nuevo` | VARCHAR(50) | |
| `observacion` | TEXT | |
| `actor_uid` | VARCHAR(100) | UID Firebase |
| `actor_rol` | VARCHAR(100) | Rol en el momento |
| `created_at` | TIMESTAMPTZ | |

### 4.5 `viv_asignaciones`

| Columna | Tipo | Descripción |
|---------|------|-------------|
| `id` | UUID | PK |
| `expediente_id` | UUID | FK UNIQUE (1:1) |
| `tipo_bien` | ENUM | `VIVIENDA` / `LOTE` |
| `identificador_bien` | VARCHAR(100) | Número de vivienda/lote |
| `domicilio_bien` | VARCHAR(300) | Dirección del bien asignado |
| `fecha_escritura` | DATE | NULL |
| `observaciones` | TEXT | |
| `created_at` | TIMESTAMPTZ | |
| `created_by` | VARCHAR(100) | |

---

## 5. Endpoints

### Beneficiarios

```
GET    /api/v1/vivienda/beneficiarios
POST   /api/v1/vivienda/beneficiarios
GET    /api/v1/vivienda/beneficiarios/{id}
PATCH  /api/v1/vivienda/beneficiarios/{id}
DELETE /api/v1/vivienda/beneficiarios/{id}
GET    /api/v1/vivienda/beneficiarios/buscar?dni={dni}
```

### Expedientes

```
GET    /api/v1/vivienda/expedientes
POST   /api/v1/vivienda/expedientes
GET    /api/v1/vivienda/expedientes/{id}
PATCH  /api/v1/vivienda/expedientes/{id}
POST   /api/v1/vivienda/expedientes/{id}/transicion
GET    /api/v1/vivienda/expedientes/{id}/historial
GET    /api/v1/vivienda/expedientes/por-programa/{programa_codigo}
```

### Programas

```
GET    /api/v1/vivienda/programas
GET    /api/v1/vivienda/programas/{id}
GET    /api/v1/vivienda/programas/{id}/estadisticas
```

### Asignaciones

```
POST   /api/v1/vivienda/asignaciones
GET    /api/v1/vivienda/asignaciones/{expediente_id}
PATCH  /api/v1/vivienda/asignaciones/{id}
```

---

## 6. Reglas de negocio

1. Un beneficiario no puede tener más de un expediente activo por programa simultáneamente.
2. El número de expediente se genera automáticamente: `VIV-{YYYY}-{NNNNNN}` (secuencial por año).
3. Solo roles `director_vivienda` y superiores pueden aprobar/rechazar expedientes.
4. La transición a `ASIGNADO` requiere que exista un registro en `viv_asignaciones`.
5. Toda transición de estado genera un registro en `viv_historial_expedientes`.
6. Los beneficiarios con `deleted_at` no pueden tener nuevos expedientes.

---

## 7. Permisos por endpoint

| Endpoint | ministro | secretario_vivienda | director_vivienda | operador_vivienda |
|----------|----------|---------------------|-------------------|-------------------|
| GET beneficiarios | ✓ lectura | ✓ | ✓ | ✓ |
| POST/PATCH/DELETE beneficiarios | ✗ | ✓ | ✓ | ✓ |
| GET expedientes | ✓ lectura | ✓ | ✓ | ✓ |
| POST expediente | ✗ | ✓ | ✓ | ✓ |
| POST transicion (aprobar/rechazar) | ✗ | ✓ | ✓ | ✗ |
| POST asignaciones | ✗ | ✓ | ✓ | ✗ |
| GET estadísticas | ✓ | ✓ | ✓ | ✓ |

---

## 8. Eventos Pub/Sub publicados

| Event type | Disparador |
|-----------|------------|
| `vivienda.beneficiario.creado` | POST beneficiarios |
| `vivienda.expediente.creado` | POST expedientes |
| `vivienda.expediente.estado_cambiado` | POST transicion |
| `vivienda.asignacion.creada` | POST asignaciones |

---

## 9. Dependencias

- `svc-notificaciones`: notificar al beneficiario ante cambios de estado (si tiene email)
- `svc-documentos`: adjuntar documentación al expediente
- UTN API: ninguna (vivienda es independiente del sistema UTN)

---

## 10. Criterios de aceptación

- [ ] CRUD completo de beneficiarios con validación de DNI único
- [ ] Máquina de estados de expedientes con transiciones validadas
- [ ] Generación automática de número de expediente
- [ ] Historial completo de cambios de estado
- [ ] Filtros por programa, estado, localidad, fecha
- [ ] Paginación en todos los listados
- [ ] Audit log en todas las escrituras
- [ ] Tests unitarios con cobertura > 80%
- [ ] Tests de integración para las transiciones de estado
- [ ] Documentación OpenAPI generada y válida
