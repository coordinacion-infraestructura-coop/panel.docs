# Organigrama Secretaría General de Gobierno — Córdoba

> Rebrand desde "Ministerio de Cooperativas y Mutuales" (ver ADR-018 en `docs/files/arquitectura.md`).
> El resto de este documento (puertos, roles, servicios `svc-auth`/`svc-notificaciones`/`svc-documentos`)
> es un boceto temprano de planificación — no refleja el estado actual implementado, ver root `CLAUDE.md`.

## Estructura jerárquica

```
Secretaría General de Gobierno
│
├── Secretaría de Vivienda
│   └── Dirección de Vivienda
│       ├── Programa Córdoba Hogar
│       ├── Programa Mi Lugar
│       └── Cordón Cuneta
│   └── Programa de Loteos
│
├── Secretaría de Gestión y Vinculación de Infraestructura
│   │   Titular: Luis Molinari
│   ├── Infraestructura Eléctrica
│   └── Agua y Saneamiento
│
├── Secretaría de Planificación y Articulación Territorial
│   │   Titular: Gabriel Fizza
│   └── Programa de Fortalecimiento para Cooperativas
│
├── Secretaría de Desarrollo
│   │   Titular: Domingo Benso
│   └── Servicio de Gestión de Cooperativas (desarrollado por UTN)
│
└── Secretaría de Infraestructura Gasífera
    ├── Programa de Conexión de Gas en Escuelas
    ├── Asesoramientos Legales y Contables
    └── Créditos para Desarrollo de Infraestructura (ejecutados por cooperativas)
│
└── Secretaría Privada Ministro
    ├── Servicio de gestión de demandas. 
```

---

## Detalle por secretaría

### 1. Secretaría de Vivienda

| Nivel | Nombre | Notas |
|-------|--------|-------|
| Dirección | Dirección de Vivienda | |
| Programa | Córdoba Hogar | Bajo Dirección de Vivienda |
| Programa | Mi Lugar | Bajo Dirección de Vivienda |
| Programa | Cordón Cuneta | Bajo Dirección de Vivienda |
| Programa | Programa de Loteos | Dependencia directa de secretaría |

**Servicio asignado**: `svc-vivienda`
**Puerto local de desarrollo**: `8001`

---

### 2. Secretaría de Gestión y Vinculación de Infraestructura

**Titular**: Luis Molinari

| Nivel | Nombre | Notas |
|-------|--------|-------|
| Área | Infraestructura Eléctrica | |
| Área | Agua y Saneamiento | |

**Servicio asignado**: `svc-infraestructura`
**Puerto local de desarrollo**: `8002`

---

### 3. Secretaría de Planificación y Articulación Territorial

**Titular**: Gabriel Fizza

| Nivel | Nombre | Notas |
|-------|--------|-------|
| Programa | Fortalecimiento para Cooperativas | |

**Servicio asignado**: `svc-territorial`
**Puerto local de desarrollo**: `8003`

---

### 4. Secretaría de Desarrollo

**Titular**: Domingo Benso

| Nivel | Nombre | Notas |
|-------|--------|-------|
| Servicio externo | Gestión de Cooperativas | Desarrollado por UTN — integración via API |

**Servicio asignado**: `svc-desarrollo`
**Puerto local de desarrollo**: `8004`
**Nota especial**: Este servicio actúa principalmente como integrador/proxy del sistema UTN. Requiere spec de integración separado.

---

### 5. Secretaría de Infraestructura Gasífera

| Nivel | Nombre | Notas |
|-------|--------|-------|
| Programa | Conexión de Gas en Escuelas | |
| Servicio | Asesoramientos Legales y Contables | |
| Programa | Créditos para Infraestructura | Ejecutados por cooperativas |

**Servicio asignado**: `svc-gasifera`
**Puerto local de desarrollo**: `8005`

---

## Servicios transversales

Estos servicios no pertenecen a ninguna secretaría específica sino que son consumidos por todas:

| Servicio | Responsabilidad | Puerto |
|----------|----------------|--------|
| `svc-auth` | Roles, permisos, tokens | 8010 |
| `svc-notificaciones` | Emails, alertas internas | 8011 |
| `svc-documentos` | Gestión documental, firma digital | 8012 |
| `svc-gateway` | API Gateway OpenAPI config | — |

---

## Mapeo roles → secretarías

| Secretaría | Rol secretario | Rol director/área |
|------------|---------------|-------------------|
| Vivienda | `secretario_vivienda` | `director_vivienda` |
| Infraestructura | `secretario_infraestructura` | `operador_infraestructura` |
| Territorial | `secretario_territorial` | `operador_territorial` |
| Desarrollo | `secretario_desarrollo` | `operador_desarrollo` |
| Gasífera | `secretario_gasifera` | `operador_gasifera` |

---

## Notas de implementación

- El sistema UTN (Secretaría de Desarrollo) es un sistema externo pre-existente. `svc-desarrollo` lo integra pero no lo reemplaza.
- Todos los programas que manejan beneficiarios deben incluir módulo de seguimiento de casos.
- Los programas de crédito (Gasífera) requieren módulo de amortización y seguimiento de cuotas.
- Asesoramientos legales y contables implica gestión de turnos y archivos adjuntos.
