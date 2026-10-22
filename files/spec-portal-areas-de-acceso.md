# Spec: Usuarios del portal — Rol + Secretarías + Áreas de acceso

**Estado**: approved
**Versión**: 1.1.0
**Servicio**: `svc-vivienda` (módulo `portal` + routers de los paneles), `svc-privada` / `svc-gasifera` / `svc-gralgob` / `svc-datos-externos` (auth), frontend (`AdminUsuariosPage`, `Layout`, `DashboardPage`)
**Responsable de spec**: Pedro Bonafe
**Última actualización**: 2026-10-21
**ADR a registrar al aprobar**: ADR-032 (§9)

### Changelog
- **1.1.0 (2026-10-21)** — implementación (§11): decisiones tomadas al implementar, sin cambiar lo
  aprobado. ADR-032 registrado.
- **1.0.0 (2026-10-21)** — aprobado. B1, B2, B4 y B6 según la propuesta. B3: Tablero de
  Gobierno y Resumen Territorial siguen visibles solo para Admin y Autoridad, sin áreas. B5: no
  se agregan áreas de función; administrar estados es por rol
  (`spec-vivienda-estados-por-area.md §4.1`).
- **0.1.0 (2026-10-21)** — versión inicial con el esquema propuesto por el usuario, ajustes y
  decisiones abiertas. Primera secretaría: Vivienda; el resto se ordena después.

---

## 0. Origen

Pedido (2026-10-21): algunos usuarios del área técnica de Vivienda ahora también tienen que ver
los paneles CC, CH y ML, cosa que el rol `TecnicoDGV` no permite (solo ve Checklist Técnico y
tableros). Se propone un esquema de tres niveles que sirva para todas las secretarías:

1. **Rol**: Admin, Supervisor, Operador, Consulta, Autoridad.
2. **Secretarías asignadas**: Vivienda, Privada, General de Gobierno, Gasífera, etc.
3. **Áreas de acceso**: casilleros que dependen de la secretaría. Para Vivienda: CC, CH, ML y
   Checklist Técnico. Restringen qué paneles ve el usuario.

## 1. Situación actual

- `portal_usuarios` (rol único, global) + `portal_usuario_secretarias` (N secretarías).
- `svc-vivienda` **no controla la secretaría**: `require_roles` solo mira el rol. Hoy un
  Operador de Privada que conozca la URL puede leer y editar los paneles de Vivienda; la
  secretaría solo se respeta en la navegación del frontend. Privada, Gasífera y Gral. de
  Gobierno sí la controlan en el backend.
- `TecnicoDGV` es un rol que en realidad expresa un **área de acceso** (solo Checklist +
  tableros), implementado con tuplas locales en cada router y filtros a mano en
  `Layout.tsx` y `DashboardPage.tsx`.

## 2. Evaluación del esquema propuesto

El esquema es correcto y resuelve el caso: separa **qué puede hacer** (rol) de **dónde**
(secretaría y áreas). Reemplaza al rol `TecnicoDGV` y evita inventar un rol nuevo por cada
combinación. Con estos ajustes:

1. **El rol es uno solo para todo.** Con el esquema tal cual, un usuario técnico que debe
   **editar** el Checklist pero solo **ver** CC/CH/ML no se puede expresar: si es Operador, edita
   los paneles; si es Consulta, no edita el Checklist. Propuesta (B1): cada área marcada lleva un
   nivel, **Ver** o **Editar**. El permiso efectivo es el menor entre el rol y el nivel del área
   (un Consulta con "Editar" sigue sin editar).
2. **El control tiene que estar en el backend, no solo en el menú.** Cada endpoint de un panel
   verifica secretaría + área. Esto cierra además el hueco de §1 (Vivienda no controla
   secretaría).
3. **Hay pantallas que no son de un panel**: Tablero de programas, Tablero analítico, Resumen
   Territorial, Informes, Informe de localidades. Ver B3.
4. **Admin** no se restringe por áreas. **Autoridad** sí (lee las áreas que tenga marcadas);
   Resumen Territorial sigue siendo transversal para Autoridad, como hoy.
5. **Usuarios existentes**: la migración les marca todas las áreas de sus secretarías con el
   nivel de su rol, para que nadie pierda acceso el día del deploy. Los `TecnicoDGV` pasan a
   Operador + Vivienda + Checklist Técnico (Editar), que es lo que tienen hoy.

## 3. Decisiones abiertas

| # | Decisión | Propuesta |
|---|---|---|
| B1 | ¿Nivel Ver / Editar por área? | Sí (§2.1). Sin esto el caso técnico no se resuelve. |
| B2 | Nombre de la UI | "Áreas de acceso". |
| B3 | Pantallas transversales de la secretaría | Tablero de programas y Tablero analítico: visibles con cualquier área de Vivienda (como hoy para `TecnicoDGV`). Informes CC/CH: atados al área CC / CH. **Resuelto (usuario):** Tablero de Gobierno y Resumen Territorial quedan como hoy, solo Admin y Autoridad, fuera del esquema de áreas. |
| B4 | ¿Se elimina el rol `TecnicoDGV`? | Sí, tras migrar a sus usuarios. Queda aceptado en lectura un tiempo para no romper sesiones abiertas. |
| B5 | ¿"Cada área edita sus propios estados" entra acá? | **Resuelto (usuario): no.** Es por rol: Operador crea/edita, Supervisor además borra (`spec-vivienda-estados-por-area.md §4.1`). Sin áreas de función. |
| B6 | Usuario sin ninguna área marcada en una secretaría | No ve nada de esa secretaría (explícito, no "todo"). |

## 4. Modelo

```
portal_usuario_areas
  email       VARCHAR(255)  NOT NULL
  secretaria  VARCHAR(100)  NOT NULL
  area        VARCHAR(50)   NOT NULL
  nivel       VARCHAR(10)   NOT NULL   -- lectura | edicion
  PK (email, secretaria, area)
  FK (email, secretaria) → portal_usuario_secretarias ON DELETE CASCADE
```

El catálogo de áreas vive en código, junto a `SECRETARIAS_VALIDAS` (`app/portal/schemas.py`),
porque cada área corresponde a pantallas y endpoints que existen en el código:

```python
AREAS_POR_SECRETARIA = {
    "vivienda": {
        "cc": "Cordón Cuneta",
        "ch": "Córdoba Hogar",
        "ml": "Mi Lugar",
        "checklist": "Checklist Técnico",
    },
    # privada, gasifera, gralgob, datos-externos: se definen en la etapa siguiente.
    # Mientras tanto, una secretaría sin áreas en el catálogo se comporta como hoy.
}
```

## 5. Backend

- **Portal**: `GET /api/v1/portal/areas` (catálogo, para el formulario de usuarios);
  `POST`/`PUT /api/v1/portal/admin/usuarios` aceptan
  `areas: [{secretaria, area, nivel}]` y validan que la secretaría esté asignada y el área
  exista en el catálogo. `GET /portal/me` devuelve `areas`. Audit log en cada cambio.
- **`AuthUser`** suma `areas: dict[(secretaria, area), nivel]`, cargado en la misma consulta.
- **Nueva dependencia** `require_area(secretaria, area, escritura: bool)` en `app/auth.py`:
  `Admin` pasa siempre; para el resto exige la secretaría, el área y, si `escritura`, nivel
  `edicion` además del rol de escritura.
- **Routers de Vivienda**: CC, CH, ML y Checklist cambian `require_roles(...)` por
  `require_area(...)` (más el rol). Desaparecen las tuplas locales `*_CHECKLIST` /
  `*_TABLERO` con `TecnicoDGV`.
- **Checklist dentro de los paneles**: la pestaña "Checklist Técnico" de CC lee el módulo de
  checklist; requiere área `checklist` o `cc` en lectura. El Estado Técnico que muestran los
  paneles (solo lectura) no requiere área `checklist`.
- **Otros servicios**: `GET /internal/portal/usuarios/{email}` (ADR-015) suma `areas`. Cambio
  aditivo; `svc-privada` lo ignora hasta que su secretaría tenga áreas en el catálogo. `GET
  /api/v1/privada/me` no cambia de forma (contrato byte-compatible, ADR-008).

## 6. Frontend

- **`AdminUsuariosPage`**: debajo de cada secretaría marcada, sus áreas con un casillero y el
  nivel (Ver / Editar). Se quita `TecnicoDGV` del selector de rol.
- **Hook compartido** `usePermisos()` en `src/shared/` (`puedeVer(sec, area)`,
  `puedeEditar(sec, area)`), que reemplaza los `canManage` / `canEdit` calculados a mano en
  cada página y los filtros por `TecnicoDGV` de `Layout.tsx` y `DashboardPage.tsx`.
- **Navegación**: cada ítem del menú declara su área; se muestra si el usuario la tiene.
- **Rutas**: entrar por URL a un panel sin área muestra "Sin acceso" (el backend igual responde
  `403`).

## 7. Migración

1. Crea `portal_usuario_areas`.
2. Para cada usuario y cada secretaría asignada con áreas en el catálogo: inserta todas las
   áreas, nivel `edicion` si el rol es Admin/Supervisor/Operador, `lectura` si es
   Consulta/Autoridad.
3. `TecnicoDGV` → rol `Operador`, Vivienda con solo `checklist` (`edicion`).
4. Audit log por cada cambio; el downgrade sale de ahí.

Resultado: nadie cambia de acceso el día del deploy, salvo el cierre del hueco de §1 (usuarios
sin Vivienda asignada que hoy podían llamar a sus endpoints).

## 8. Fuera de alcance

- Áreas de Privada, Gasífera, Gral. de Gobierno y Datos Externos (etapa siguiente).
- Rol distinto por secretaría (el nivel por área cubre el caso conocido).
- Filtrar filas dentro de un panel (por localidad, departamento, etc.).

## 9. ADR-032 (texto propuesto)

**Decisión**: los permisos del portal tienen tres niveles: rol (qué acciones), secretarías
(dónde) y áreas de acceso por secretaría con nivel Ver / Editar (qué paneles). Se controla en el
backend de cada servicio. El rol `TecnicoDGV` se elimina y se expresa como área. Reemplaza la
convención de tuplas locales de `spec-checklist-tecnico-dgv.md §8`.
**Razón**: el área técnica necesita combinaciones (editar el Checklist, ver los paneles) que un
rol por módulo no puede expresar sin multiplicar roles.
**Consecuencia**: un endpoint nuevo de panel debe declarar su área. `svc-vivienda` pasa a
controlar la secretaría, cosa que hoy no hace.

## 10. Pruebas

- `require_area`: Admin pasa; sin secretaría `403`; sin área `403`; área en `lectura` + `PATCH`
  `403`; Consulta con área en `edicion` + `PATCH` `403`.
- Migración: usuario Operador con Vivienda → 4 áreas en `edicion`; `TecnicoDGV` → Operador +
  `checklist`.
- Portal: área de una secretaría no asignada → `422`; área inexistente → `422`.
- Frontend: `npm run build` + verificación manual con un usuario de cada perfil.

## 11. Implementación (2026-10-21)

Rama `portal-areas-de-acceso` en `panel.backend`, `panel.front` y `gestor.infra`.

- **Migración `0045`** (revisa a la `0044` de `spec-vivienda-estados-por-area.md`: se despliega después).
  Crea `portal_usuario_areas` y la completa según §7. Imprime los usuarios que dejan de poder leer
  los paneles de Vivienda (rol de lectura sin la secretaría y sin ser de Infraestructura/Supervisión):
  **revisar esa lista contra producción antes de desplegar**. Verificada contra Postgres (subida y
  bajada, con TecnicoDGV con y sin Vivienda).
- **FK**: solo a `portal_usuarios.email` (no compuesta a `portal_usuario_secretarias` como decía §4).
  La coherencia la mantienen el router (valida que la secretaría esté asignada) y
  `repository.update_usuario` (al quitar una secretaría borra sus áreas).
- **Backend**: `require_area(*areas)` a nivel de router (lectura) y `require_edicion((area,), *roles)`
  en cada escritura de CC/CH/ML/Checklist. Lectura del Checklist: áreas `checklist`, `cc`, `ch` o `ml`.
  Tablero de programas, Tablero analítico, Informe de localidades y núcleo genérico (programas,
  beneficiarios, expedientes, asignaciones): cualquier área de Vivienda; las escrituras del núcleo
  genérico siguen solo por rol. `geo`, `resumen_territorial`, `notificaciones`, `portal` e `internal`
  no cambian.
- **Excepciones que preservan el acceso de hoy**:
  - Infraestructura y Supervisión siguen leyendo CC/CH/ML (`lectura_comunicaciones`), porque cargan
    comunicaciones en esos paneles.
  - `Autoridad` sigue viendo Tablero de programas, Tablero analítico e Informe de localidades sin
    tener la secretaría (`abierto_a_autoridad`; lo exigía `spec-vivienda-tablero-analitico.md`).
- **Endpoint nuevo** `GET /api/v1/portal/areas` → requiere **config nueva del gateway**. Sin ella,
  la pantalla de usuarios no muestra los casilleros de áreas (el resto funciona).
- **Auditoría**: alta y edición de usuarios dejan fila en `viv_audit_log` (`portal_usuario`).
- **Frontend**: `src/shared/utils/permisos.ts` (`nivelDeArea`, `puedeEditarArea`, `puedeVerRuta`,
  `AREA_DE_RUTA`); `ProtectedRoute` acepta `requiredArea`; sin acceso redirige al inicio (como el resto
  de las rutas protegidas) en vez de una pantalla "Sin acceso". Menú (`Layout`), tarjetas del inicio
  (`DashboardPage`), enlaces de indicadores del Resumen Territorial y botones de CC/CH/ML/Checklist
  usan las áreas. `AdminUsuariosPage`: casilleros por área con nivel Ver/Editar; se quita `TecnicoDGV`.
- **Pruebas**: `tests/test_portal_areas_de_acceso.py` (17) + suite completa (506 passed);
  `npm run build` OK. Sin verificación visual todavía.
