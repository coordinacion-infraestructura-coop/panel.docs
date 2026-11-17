# Spec: Administración delegada por secretaría — usuarios y catálogos del Checklist (Vivienda)

**Estado**: approved
**Versión**: 1.0.0
**Servicio**: `svc-vivienda` (módulos `portal` y `checklist_tecnico`), frontend (`AdminUsuariosPage`, `AdminCatalogosChecklistPage`, `Layout`, `App.tsx`)
**Responsable de spec**: Pedro Bonafe
**Última actualización**: 2026-11-17
**Modifica**: `spec-portal-areas-de-acceso.md` (§3 B5, §4, §5) y `spec-checklist-tecnico-dgv.md` (§8, catálogos `/admin/...`)
**ADR**: ADR-035 (registrado en `arquitectura.md`)

### Changelog
- **1.0.0 (2026-11-17)** — aprobado por el usuario. D1: se acepta el área de función `usuarios`
  como excepción explícita a B5. D2: el delegado **no** puede otorgar el área `usuarios`; solo el Admin
  habilita administradores. ADR-035 registrado.
- **0.2.0 (2026-11-17)** — decisiones del usuario: (1) el Supervisor delegado **puede** crear
  otros Supervisores; (2) los catálogos del Checklist los administra **cualquier Supervisor con el
  área Checklist en Editar**; (3) los usuarios con más de una secretaría siguen siendo **solo del
  Admin**. D1 y D2 quedaban abiertas (§3).
- **0.1.0 (2026-11-17)** — análisis inicial.

---

## 0. Origen

Pedido del usuario (2026-11-17): (a) el Supervisor de Vivienda necesita editar los estados del
Checklist Técnico; (b) Vivienda necesita dar de alta sus propios usuarios, sin depender del Admin,
y sin poder afectar a otras secretarías.

## 1. Situación actual

| Acción | Hoy | Dónde |
|---|---|---|
| Cambiar el estado del expediente, ítems, hitos, etc. | Admin/Supervisor/Operador **con área `checklist` en Editar** | `checklist_tecnico/router.py` (`ESCRITURA_CHECKLIST`) |
| Crear o editar los catálogos del Checklist (estado del expediente, repartición, estado de ítem) | **Solo Admin** | `checklist_tecnico/router.py` (`/checklist-tecnico/admin/...`, `ROLES_ADMIN`); `/admin/catalogos-checklist` en `App.tsx` |
| Alta, edición y baja de usuarios | **Solo Admin** | `portal/router.py` (`_require_admin`); `/admin/usuarios` en `App.tsx` |

Lo primero ya funciona: un Supervisor que no puede cambiar el estado de un expediente tiene el
área `checklist` en "Ver" o no la tiene, y se corrige desde el panel de usuarios.

Condición importante: `portal_usuarios` es **una sola tabla para todo el portal**. `svc-privada`,
`svc-gasifera`, `svc-gralgob` y `svc-datos-externos` la leen a través de
`GET /internal/portal/usuarios/{email}` (ADR-015). Delegar la administración exige que el
**backend** limite qué filas toca cada administrador delegado. Filtrar solo en la pantalla no
alcanza.

## 2. Alcance

### Incluido
1. Catálogos del Checklist abiertos al Supervisor con área `checklist` en Editar.
2. Administración delegada de usuarios **de Vivienda** por los Supervisores que el Admin habilite.

### Fuera de alcance
- Otras secretarías. El esquema queda listo para ellas: cada una suma su área `usuarios` al
  catálogo cuando lo pida.
- Borrar usuarios. El delegado solo **desactiva** (`activo = false`); el borrado sigue siendo del
  Admin, igual que `ROLES_ELIMINACION = ("Admin",)`.
- Usuarios con más de una secretaría (decisión 3 del usuario).

## 3. Decisiones

| # | Decisión | Resolución |
|---|---|---|
| U1 | ¿El delegado puede crear Supervisores? | **Sí** (usuario, 2026-11-17). |
| U2 | ¿Quién administra los catálogos del Checklist? | **Supervisor con `checklist` en Editar** (usuario, 2026-11-17). No hace falta un área nueva. |
| U3 | Usuarios con varias secretarías | **Solo Admin** (usuario, 2026-11-17). |
| D1 | **Conflicto con `spec-portal-areas-de-acceso.md` B5** ("sin áreas de función"). La habilitación para administrar usuarios se modela como un área `usuarios` en el catálogo de Vivienda. Es un área de función, no de panel. | **Resuelto (usuario, 2026-11-17): se acepta la excepción.** B5 resolvía otra cosa (quién edita los estados de cada panel). Un área evita una migración y reusa toda la maquinaria de ADR-032 (casillero en el formulario, `nivel_de_area`, `/portal/me`). La alternativa es una columna `administra` en `portal_usuario_secretarias`, que necesita migración y lógica propia. |
| D2 | ¿Un delegado puede **otorgar** el área `usuarios`, es decir, crear a otro administrador delegado? | **Resuelto (usuario, 2026-11-17): no**, solo el Admin. Como U1 permite crear Supervisores, si el delegado pudiera además darles `usuarios`, la cantidad de administradores crecería sin control del Admin. Con esta regla, el delegado crea Supervisores "comunes" y el Admin decide quién administra. |

## 4. Modelo

Sin migración. El catálogo en código (`portal/schemas.AREAS_POR_SECRETARIA`) suma un área:

```python
"vivienda": {
    "cc": "Cordón Cuneta",
    "ch": "Córdoba Hogar",
    "ml": "Mi Lugar",
    "checklist": "Checklist Técnico",
    "usuarios": "Administración de usuarios",   # nuevo
},
```

- **Administrador delegado de Vivienda** = rol `Supervisor` + secretaría `vivienda` + área
  `usuarios` en `edicion`. Rige la regla de ADR-032 (el menor entre rol y área): un Operador o un
  Consulta con `usuarios` en Editar **no** administra.
- La migración `0045` no se repite: nadie recibe el área `usuarios` automáticamente, el Admin la
  asigna a mano.
- `usuarios` no da acceso a ningún panel. `require_area()` sin áreas ("cualquier área de
  Vivienda", usado por los tableros) la ignora, para que un Supervisor que solo tenga `usuarios`
  no vea los tableros por eso.

## 5. Backend

### 5.1 Catálogos del Checklist

Los seis endpoints `POST`/`PATCH /api/v1/vivienda/checklist-tecnico/admin/{estado-expediente,reparticion,item-estado}`
cambian `require_roles(*ROLES_ADMIN)` por:

```python
ADMIN_CATALOGOS_CHECKLIST = require_edicion(("checklist",), "Admin", "Supervisor")
```

Sin cambios de forma ni de gateway (los paths ya están publicados). Audit log como hoy.
Consecuencia a tener presente: el catálogo de estado del expediente es **único para CC, CH y ML**
y alimenta el "Estado Técnico" de los tres paneles (`spec-estado-tecnico-desde-checklist.md`).
Un cambio impacta a todo Vivienda.

### 5.2 Usuarios

Los mismos endpoints `/api/v1/portal/admin/usuarios[/{email}]` (sin paths nuevos, sin cambio de
gateway). `_require_admin` se reemplaza por `_require_administrador`, que devuelve el **alcance**
del actor:

- `Admin` → alcance total, comportamiento actual sin cambios.
- `Supervisor` con `vivienda/usuarios` en `edicion` → alcance `{"vivienda"}`.
- Cualquier otro → `403 PERMISO_INSUFICIENTE`.

Reglas para el actor con alcance `{"vivienda"}`:

| Operación | Regla |
|---|---|
| `GET /admin/usuarios` | El **servidor** devuelve solo usuarios cuyas secretarías sean exactamente `["vivienda"]`. |
| `GET /admin/usuarios/{email}` | `404` si el usuario no está en su alcance (no revela que existe). |
| `POST` | `secretarias` tiene que ser `["vivienda"]` (`422 SECRETARIA_FUERA_DE_ALCANCE`). `rol` ∈ {`Supervisor`, `Operador`, `Consulta`} (`422 ROL_FUERA_DE_ALCANCE`). Áreas solo de Vivienda y sin `usuarios` (D2): `422 AREA_FUERA_DE_ALCANCE`. Si el email ya existe **en cualquier secretaría**: `409` con un mensaje genérico ("Ese email ya está registrado; pedí el alta a un administrador"), sin decir dónde. |
| `PUT` | El usuario destino tiene que estar en su alcance (si no, `404`). Mismas restricciones de rol, secretarías y áreas que `POST`. No puede cambiar sus propios datos (`400 AUTO_EDICION`). No puede editar ni desactivar a otro usuario que tenga `usuarios` (un delegado no degrada a otro delegado; eso lo hace el Admin). `activo=false` permitido. |
| `DELETE` | `403`. Solo Admin. |

Excluidos siempre para el delegado: los roles `Admin` y `Autoridad`, y las secretarías
`infraestructura` y `supervision` (dan lectura a CC/CH/ML por la excepción de comunicaciones).

Audit log como hoy (`viv_audit_log`, `resource_type="portal_usuario"`), con el actor delegado.
`GET /internal/portal/usuarios/{email}` y `GET /portal/me` no cambian.

## 6. Frontend

- `permisos.ts`: `esAdministradorDe(user, 'vivienda')` = rol `Admin`, o rol `Supervisor` con
  `puedeEditarArea(user, 'vivienda', 'usuarios')`.
- **`/admin/usuarios`**: la ruta se abre a Admin o administrador delegado. La misma
  `AdminUsuariosPage` en "modo secretaría" cuando el actor no es Admin: secretaría fija y oculta
  (Vivienda), selector de rol con Supervisor/Operador/Consulta, solo las áreas de Vivienda (sin
  `usuarios`, D2), sin botón Eliminar (sí Desactivar), y un aviso de que se listan solo los
  usuarios exclusivos de Vivienda.
- **`/admin/catalogos-checklist`**: la ruta y el enlace del menú se abren a Admin o Supervisor con
  `checklist` en Editar. Se cambia el texto de las insignias "⚙ admin" de
  `ChecklistTecnicoPage.tsx` ("Catálogo administrable por un usuario Admin").
- `Layout.tsx`: el menú de administración muestra cada entrada según el permiso correspondiente,
  no "solo Admin".

## 7. Despliegue

Solo `svc-vivienda` y el frontend. Sin migración, sin config nueva del gateway. Después del
despliegue, el Admin asigna `usuarios` (Editar) a los Supervisores de Vivienda que correspondan y
verifica que tengan `checklist` en Editar.

## 8. Pruebas

- Catálogos: Supervisor + `checklist` Editar → `201`/`200`; Supervisor + `checklist` Ver →
  `403`; Operador + `checklist` Editar → `403`; Admin → `201`.
- Usuarios (delegado): lista sin usuarios de otras secretarías ni mixtos; `GET` de un usuario mixto
  → `404`; alta con `privada` → `422`; alta con rol `Admin`/`Autoridad` → `422`; alta de Supervisor
  → `201`; alta con área `usuarios` → `422` (D2); email existente en Privada → `409` genérico;
  editar a un usuario mixto → `404`; editarse a sí mismo → `400`; editar a otro delegado → `403`;
  `DELETE` → `403`; Supervisor sin `usuarios` → `403`; Operador con `usuarios` Editar → `403`.
- `require_area()` sin áreas: un Supervisor con solo `usuarios` no ve los tableros.
- Admin: toda la suite de `test_portal_areas_de_acceso.py` sin cambios.
- Frontend: `npm run build` + verificación manual con un Admin, un delegado y un Supervisor común.

## 9. ADR-035 (texto propuesto)

**Decisión**: la administración de usuarios puede delegarse por secretaría. Un `Supervisor` con
el área `usuarios` de su secretaría en Editar da de alta, edita y desactiva usuarios **exclusivos
de esa secretaría**, con roles hasta Supervisor. El backend limita el alcance; los usuarios con
varias secretarías, el borrado y la habilitación de nuevos administradores quedan para el Admin.
Los catálogos del Checklist Técnico pasan de solo Admin a Supervisor con `checklist` en Editar.
Primera secretaría: Vivienda.
**Razón**: pedido del usuario (2026-11-17). Vivienda necesita autonomía para sus altas sin poder
afectar a otras secretarías, que comparten la tabla `portal_usuarios` (ADR-003/015).
**Consecuencia**: amplía ADR-032 con un área de función (`usuarios`), excepción explícita a la
decisión B5 de `spec-portal-areas-de-acceso.md`. El alcance se verifica en el backend en cada
operación del portal.
