# Spec: Mi Lugar — Valor Fiscal sugerido y valor de referencia de mercado desde IDECOR

**Estado**: approved
**Versión**: 1.1.0
**Responsable de spec**: Pedro Bonafe
**Última actualización**: 2026-10-19
**Servicios**: `svc-datos-externos` (consulta a IDECOR) · `svc-vivienda` (`mi_lugar`) · `frontend` (`MiLugarPage.tsx`) · `infra` (gateway + Cloud Scheduler)

Origen: relevamiento de fuentes externas del 2026-10-19 (Estadística Córdoba, IDECOR, Poblaciones).
Es el primer paso concreto hacia la cadena `valor_fiscal → valor_expropiacion` de Mi Lugar, que hoy
no está implementada en el servidor (los campos se guardan tal como los manda el cliente).
Decisión de arquitectura: ADR-031.

## Changelog

- **1.1.0** (2026-10-19): **desplegado en producción.** Migración `0043` aplicada; `svc-vivienda`
  y `svc-datos-externos` en la revisión `cbc1f39` (`VALUACION_IDECOR_ENABLED=true`); gateway
  `ministerio-config-v20261019`; Cloud Scheduler `mi-lugar-valuacion-idecor` (lunes 06:00,
  America/Argentina/Cordoba); frontend en Firebase Hosting (`ee7f09d`). Primera corrida: 21
  proyectos de Expropiación → 16 `ok`, 1 `sin_parcelas` (Capital - San Felipe: el punto no cae
  en una parcela), 4 `sin_datos` (sin puntos ni nomenclaturas). **5 coinciden al centavo** con el
  `valor_fiscal` del área (Villa María, Estación Juárez Celman, Las Higueras, Río Cuarto,
  Salsipuedes); Almafuerte (−60%), Montecristo (+10%) y Cruz del Eje (−2%) difieren, lo que
  sugiere parcelas faltantes o de más (se completan con nomenclaturas). 8 proyectos de Capital no
  tienen `valor_fiscal` cargado y ahora tienen sugerencia. Capital - Ferreyra: la parcela tiene
  valuación $0 en Catastro. Criterios §8 cumplidos: 1, 2 (ninguna escritura del scheduler en
  `ml_proyecto`), 5. Pendientes: 3 (prueba con la fuente apagada en producción, no ejecutada), 4
  (unidad rural) y 6 (verificación visual del área). El estado pasa a `implemented` al cerrarlos.
- **1.0.1** (2026-10-19): corrección de la cuenta de ejemplo de §3 (valor de mercado de la parcela
  de Villa María: $17.154.786.999). Código escrito en la rama `mi-lugar-valuacion-idecor` de
  `services/`, `frontend/` e `infra/`, sin desplegar; runbook en
  `infra/deploy-mi-lugar-valuacion-idecor.md`. El estado sigue en `approved` hasta el deploy.
- **1.0.0** (2026-10-19): **aprobado por el usuario**, con las respuestas a §9 de la v0.1.0:
  1. el valor sugerido es la valuación **total** (tierra + mejoras);
  2. las parcelas salen de **los puntos y de una lista de nomenclaturas** que carga el área (la
     lista entra en esta versión, no en una v2);
  3. **solo Expropiación**; Muni/Prov se agregan si el área lo pide;
  4. "coincide" = diferencia menor a $1;
  5. se registra ADR-031.

  Se suma por pedido del usuario una columna informativa **"Valor referencia de mercado"** con el
  mapa de valores de la tierra de IDECOR (§4.2). Agregado en la redacción: la capa urbana no tiene
  valor para parcelas rurales, así que se usa también la capa rural equivalente (2 de los 3 proyectos
  validados son rurales).
- **0.1.0** (2026-10-19): primera versión, en draft. Incluye la validación contra datos reales (§3).

---

## 1. Propósito

En el panel de Mi Lugar (pestaña Expropiación) se suman dos columnas de **solo lectura**, que
vienen de IDECOR y sirven como referencia:

1. **"Valor Fiscal (IDECOR)"**: la valuación fiscal vigente que publica Catastro para las parcelas
   del proyecto. Es una **sugerencia** para la columna del área.
2. **"Valor referencia de mercado"**: el valor de la tierra según el mapa de valores de IDECOR. Es
   **solo informativa**.

La columna **`valor_fiscal` del área no cambia**: sigue siendo de carga manual, la edita el área y
es el único valor que usa el sistema. Ningún valor de IDECOR **se escribe solo** sobre
`valor_fiscal`. El área puede copiar la sugerencia fiscal con una acción explícita (§6.2), que queda
auditada como cualquier edición. El valor de mercado no se puede copiar a ningún campo.

## 2. Alcance

### Incluido

- Lista editable de **nomenclaturas catastrales** por proyecto (§5.2, §6.2).
- Las parcelas del proyecto son la unión, sin repetir, de:
  - las que contienen cada punto de `viv_ml_geo_puntos`;
  - las de la lista de nomenclaturas.
- Para esas parcelas: suma de la valuación fiscal y suma del valor de referencia de mercado.
- Guardado del último resultado por proyecto en `db_vivienda` (§5.2), con fecha de consulta,
  vigencia de la valuación, año del mapa de valores y detalle por parcela.
- Actualización:
  - automática, al crear un proyecto o cambiar sus puntos o nomenclaturas;
  - semanal, por Cloud Scheduler;
  - manual, con un botón "Recalcular" (§5.4).
- Las dos columnas en la tabla, la sugerencia y la lista de nomenclaturas en el modal de edición, y
  las columnas en el Excel (§6).

### Fuera de alcance

- Calcular `valor_expropiacion` o cualquier otro campo derivado.
- Cambiar `valor_fiscal`: ni su tipo, ni su validación, ni quién lo edita.
- Tipos `muni` y `prov`: se agregan si el área lo pide.
- Usar el valor de mercado en algún cálculo.

## 3. Validación contra datos reales (2026-10-19)

Se consultó la capa `idecor:parcelas` con los puntos de tres proyectos de `seed_data.py`, y se
sumó `Valuacion` (tierra + mejoras) de las parcelas encontradas:

| Proyecto | `valor_fiscal` del área | Suma IDECOR | Parcelas | Resultado |
|---|---|---|---|---|
| Villa María | 3.848.546.670,36 | 3.848.546.670,36 | 2 urbanas, baldías | **Coincide al centavo** |
| Estación Juárez Celman | 2.238.814.197,52 | 2.238.814.197,52 | 2 rurales (1 edificada) | **Coincide al centavo** |
| Almafuerte | 2.016.391.989,35 | 816.433.001,32 | 4 rurales | No coincide: los puntos no cubren todas las parcelas (la observación del proyecto dice "Ver q parcelas quedan") |

Conclusiones:

1. El área ya carga como `valor_fiscal` la **valuación fiscal total de Catastro** (tierra +
   mejoras) de las parcelas. La sugerencia mide lo mismo.
2. Con solo los puntos, la sugerencia depende de que haya **un punto por parcela**. Por eso entra la
   lista de nomenclaturas (§5.2), y la UI muestra cuántas parcelas encontró y cuáles (§6.1).
3. La superficie de las parcelas no coincide con `superficie` (Villa María: 44,79 ha en parcelas
   contra 105,21 ha declaradas). No se usa ni se compara en esta versión.

Valor de mercado para la parcela de Villa María `1604220301275100` (29,74 ha urbanas):
57.687 $/m² × 297.377 m² = $17.154.786.999. Su valuación fiscal de la tierra es $2.401.670.179,86.

## 4. Fuente

### 4.1 IDECOR — capa `idecor:parcelas` (valor fiscal)

- WFS público, sin autenticación:
  `https://idecor-ws.mapascordoba.gob.ar/geoserver/idecor/wfs`. 1.957.985 parcelas de toda la
  provincia.
- Sistema de coordenadas **EPSG:22174** (Gauss-Krüger faja 4). Los puntos de Mi Lugar están en
  WGS84 y hay que reproyectarlos (`pyproj`). Un filtro con `SRID=4326` en el CQL **no devuelve
  nada** (probado), así que se reproyecta siempre en el cliente.
- Consultas probadas (~0,2 s cada una):
  - Por punto: `typeName=idecor:parcelas&outputFormat=application/json&propertyName=…&CQL_FILTER=INTERSECTS(geom,POINT(x y))`
  - Por nomenclatura: `CQL_FILTER=Nomenclatura='1604220301275100'`
- Campos que se usan:

| Campo IDECOR | Uso |
|---|---|
| `Nomenclatura` | Identifica la parcela; evita contarla dos veces |
| `Nro_Cuenta` | Cuenta de Rentas; se muestra en el detalle |
| `Tipo_Parcela`, `Estado`, `desig_oficial`, `localidad` | Detalle (urbana/rural, baldío/edificado) |
| `Valuacion` | **Valor fiscal sugerido** (suma) |
| `Valuacion_Tierra_Urbana` + `Valuacion_Tierra_Rural`, `Valuacion_Mejoras` | Desglose en el detalle |
| `Superficie_Tierra_Urbana` + `Superficie_Tierra_Rural` | Detalle (m²) y base del valor de mercado |
| `vigencia_desde` | Vigencia de la valuación (las de 2026 rigen desde el 05/06-11-2025) |
| geometría | Punto interior de la parcela, para consultar el valor rural (§4.2) |

### 4.2 IDECOR — mapa de valores de la tierra (valor de referencia de mercado)

Son dos capas por año. **No son la valuación fiscal**: la misma parcela de Villa María tiene
57.687 $/m² en el mapa de valores 2025 y 40.380,9 $/m² en su valuación fiscal.

| Parcela | Capa | Campo | Unidad | Valor de mercado de la parcela |
|---|---|---|---|---|
| Urbana | `valor_tierra_urbana_AAAA` (polígono por parcela, con `nomenclatura`) | `vut_pesos` | $/m² | `vut_pesos × superficie_terreno` |
| Rural | `valor_tierra_rural_AAAA` (polígono por zona de valor, sin nomenclatura) | `vur_pesos` | $/ha | `vur_pesos × superficie rural en ha` |

- **Año**: se usa el más reciente publicado. Se averigua leyendo `GetCapabilities` y tomando el
  mayor `AAAA` que tenga las dos capas (hoy: **2025**; `…_2026` no existe todavía). El año usado se
  guarda y se muestra.
- **Parcela urbana**: se busca por `nomenclatura` en la capa urbana. Si no aparece o `vut_pesos`
  viene vacío, la parcela queda sin valor de mercado.
- **Parcela rural**: en la capa urbana figura con `vut_pesos` vacío, así que se consulta la capa
  rural por el punto interior de la parcela. Si la parcela cruza dos zonas de valor, se toma la del
  punto interior: es una referencia, no un cálculo exacto.
- **Unidad rural**: el $/ha se dedujo del orden de magnitud contra la valuación fiscal (Juárez
  Celman: 53.566.500 $/ha de mercado contra 35,9 M$/ha fiscal; Almafuerte: 9.614.500 contra
  7,27 M$/ha), porque la capa no declara la unidad. **Se confirma con IDECOR o el área antes del
  deploy** (§8).
- Parcela mixta (con superficie urbana y rural): se suman las dos partes.

### 4.3 Lo que la fuente no garantiza

No tiene SLA, el texto puede llegar con la codificación rota (hay que leer los bytes como UTF-8) y
el esquema puede cambiar sin aviso. Si falta un campo esperado, la consulta falla entera (§5.3).

## 5. Backend

### 5.1 Dónde vive cada cosa

- **La consulta a IDECOR vive en `svc-datos-externos`** (ADR-031, amplía ADR-025/ADR-028): las
  integraciones externas frágiles quedan aisladas de `svc-vivienda`. Endpoint IAM-only, sin
  estado: `POST /internal/datos-externos/catastro/valuacion`.
  - Entrada: `{ "puntos": [{ "lat": ..., "lng": ... }], "nomenclaturas": ["..."] }` (máximo 50 de
    cada uno; al menos un elemento en total).
  - Salida:
    - las parcelas, sin repetir por `Nomenclatura`, con los campos de §4.1 y §4.2 y su origen
      (`punto`, `nomenclatura` o `ambos`);
    - los totales: valor fiscal, tierra, mejoras y valor de mercado;
    - la vigencia más antigua y el año del mapa de valores;
    - los puntos que no cayeron en ninguna parcela;
    - las nomenclaturas que no existen en IDECOR.
- **El resultado se guarda en `svc-vivienda`**, junto al proyecto. Así el panel, que carga todo de
  una vez y filtra en el cliente, no hace llamadas entre servicios al listar.
- Grant: `svc-vivienda@` ya tiene `run.invoker` sobre `svc-datos-externos` (ADR-025). No hace falta
  uno nuevo.

### 5.2 Modelo (`svc-vivienda`, migración nueva)

**`viv_ml_nomenclaturas`**: la lista que carga el área.

| Columna | Tipo | Nota |
|---|---|---|
| `id` | `VARCHAR(36)` PK | uuid |
| `proyecto_id` | `VARCHAR(36)` FK `viv_ml_proyectos.id` `ON DELETE CASCADE` | |
| `nomenclatura` | `VARCHAR(30)` | Solo dígitos (se quitan espacios, puntos y guiones al guardar) |
| `created_at`, `created_by` | | |
| `deleted_at` | `TIMESTAMPTZ` NULL | Soft delete |

Índice único parcial `(proyecto_id, nomenclatura) WHERE deleted_at IS NULL`. Se edita como
`geo_puntos`: el PATCH del proyecto recibe la lista completa y el servicio la reconcilia, con soft
delete de las que salen.

**`viv_ml_valuacion_idecor`**: una fila por proyecto, que se reemplaza en cada consulta buena.

| Columna | Tipo | Nota |
|---|---|---|
| `proyecto_id` | `VARCHAR(36)` PK, FK `viv_ml_proyectos.id` `ON DELETE CASCADE` | |
| `estado` | `VARCHAR(20)` | `ok` · `parcial` (algún punto sin parcela o nomenclatura inexistente) · `sin_datos` (sin puntos ni nomenclaturas) · `sin_parcelas` · `error` |
| `valor_total` | `NUMERIC(18,2)` NULL | Suma de `Valuacion` |
| `valor_tierra` | `NUMERIC(18,2)` NULL | |
| `valor_mejoras` | `NUMERIC(18,2)` NULL | |
| `valor_mercado` | `NUMERIC(18,2)` NULL | Suma de §4.2; NULL si ninguna parcela tiene valor |
| `mercado_anio` | `INTEGER` NULL | Año del mapa de valores |
| `mercado_parcelas_sin_valor` | `INTEGER` | |
| `cant_parcelas` | `INTEGER` | |
| `puntos_sin_parcela` | `INTEGER` | |
| `nomenclaturas_no_encontradas` | `TEXT` (JSON) | Lista |
| `vigencia_desde` | `DATE` NULL | |
| `parcelas` | `TEXT` (JSON) | Detalle de §4.1/§4.2, para el tooltip |
| `consultado_at` | `TIMESTAMPTZ` | Última consulta buena |
| `ultimo_error` | `TEXT` NULL | |
| `ultimo_error_at` | `TIMESTAMPTZ` NULL | |

Se usa una tabla aparte, y no columnas en `viv_ml_proyectos`, porque es un dato de otra fuente, con
su propio ciclo de vida, y no debe mezclarse con los campos que edita el área.

Un error **no pisa** un resultado bueno anterior: solo actualiza `ultimo_error`/`ultimo_error_at`,
y la UI sigue mostrando el último valor bueno con su fecha. Si nunca hubo resultado bueno, la fila
queda con `estado = error`.

### 5.3 Comportamiento ante fallas (fail-open)

- Si IDECOR o `svc-datos-externos` no responden, el alta y la edición del proyecto **se guardan
  igual**: la consulta corre en segundo plano y no puede hacer fallar un PATCH.
- La sugerencia queda como "no disponible", con la fecha del último intento.
- `svc-datos-externos` responde 502 si falla la consulta entera (mismo criterio que
  `SheetReadError` en los syncs), y 200 con `puntos_sin_parcela` /
  `nomenclaturas_no_encontradas` si solo falla algún elemento.
- Si falla solo el mapa de valores, el valor fiscal se devuelve igual y el de mercado queda NULL.

### 5.4 Cuándo se actualiza

1. **Al crear un proyecto o cambiar sus `geo_puntos` o nomenclaturas**: tarea en segundo plano
   después del commit, solo si `tipo = 'exp'`.
2. **Semanal**: Cloud Scheduler → `POST /internal/mi-lugar/valuacion-idecor/actualizar` (IAM-only,
   montado sin `/api/v1`, mismo patrón que el resto de `internal`). Recorre los proyectos
   `tipo = 'exp'` no borrados. Catastro revalúa una vez por año (noviembre) y el mapa de valores
   sale una vez por año. La frecuencia semanal capta los dos cambios sin cargar la fuente.
3. **Manual**: `POST /api/v1/vivienda/mi-lugar/proyectos/{id}/valuacion-idecor`, para los roles de
   escritura de Mi Lugar. Recalcula y devuelve el resultado. **Necesita una config nueva del
   gateway**, con su `options:` sin seguridad para CORS.

### 5.5 API de lectura y escritura

- `ProyectoMLOut` suma:
  - `nomenclaturas: list[str]`;
  - un objeto opcional `valuacion_idecor`, con los campos de §5.2 y `parcelas` /
    `nomenclaturas_no_encontradas` ya como listas.
- No hay `GET` nuevo: los datos llegan con `GET /mi-lugar/proyectos` y
  `GET /mi-lugar/proyectos/{id}`.
- `ProyectoMLCreate` / `ProyectoMLUpdate` suman `nomenclaturas: list[str]` (opcional en el update:
  `None` = no tocar). Máximo 50 por proyecto.
- Auditoría:
  - Los cambios de nomenclaturas van en el audit log del PATCH, como el resto del proyecto.
  - El recálculo manual escribe en el audit log (`accion = "valuacion_idecor_recalcular"`).
  - Los recálculos automáticos no, para no llenar el log cada semana; queda `consultado_at`.

## 6. Frontend (`MiLugarPage.tsx`, pestaña EXP)

### 6.1 Tabla

- Dos columnas nuevas a la derecha de "Valor Fiscal", en el mismo grupo financiero y solo en la
  pestaña `exp`:
  - **"Valor Fiscal (IDECOR)"**;
  - **"Valor referencia de mercado"**.
- Estilo de dato externo: texto gris/itálica, para que no se confundan con el dato del área.
- "Valor Fiscal (IDECOR)" lleva una marca de comparación contra `valor_fiscal`:
  - **"= coincide"** si la diferencia es menor a $1;
  - **"Δ +x% / −x%"** si difieren;
  - nada si `valor_fiscal` está vacío.
- "Valor referencia de mercado" muestra debajo el año del mapa de valores. Si hay parcelas sin
  valor, muestra además "N parcelas sin valor". No lleva marca de comparación.
- Tooltip, común a las dos columnas:
  - vigencia, fecha de consulta y año del mapa de valores;
  - una línea por parcela: nomenclatura, origen, tipo, estado, superficie, valuación y valor de
    mercado;
  - "N puntos sin parcela" y "Nomenclatura X no encontrada" si `estado = parcial`.
- Estados sin valor:
  - "sin puntos ni nomenclaturas" (`sin_datos`);
  - "sin parcelas en IDECOR";
  - "no disponible" (`error`, sin valor bueno anterior).
- Las dos columnas se pueden ordenar, como el resto de las columnas de montos.

### 6.2 Modal de edición (alta y edición)

- Campo nuevo **"Nomenclaturas catastrales"**, junto a las coordenadas: una por línea, mismo
  formato de textarea que los puntos.
- Debajo del campo "Valor Fiscal ($)", solo en edición:

  > Sugerencia IDECOR: $ 3.848.546.670,36 · 2 parcelas · vigencia 06/11/2025 · [Usar este valor] [Recalcular]
  > Referencia de mercado (IDECOR 2025): $ … (informativo)

  - **"Usar este valor"** copia el valor fiscal sugerido al campo del formulario y **no guarda**:
    el área guarda con el botón del modal, como siempre. Es la única forma en que un valor de
    IDECOR llega a `valor_fiscal`. El valor de mercado no tiene este botón.
  - "Recalcular" llama al endpoint manual de §5.4 y refresca los dos valores.
- Permisos: los mismos que ya tiene el modal (`canEdit`). Las columnas y el tooltip los ven todos
  los que ven el panel.

### 6.3 Excel

La exportación suma, al lado de "Valor Fiscal ($)":

- "Valor Fiscal IDECOR ($)" y "Vigencia IDECOR";
- "Valor referencia de mercado ($)" y "Año mapa de valores";
- "Nomenclaturas", separadas por `|`.

## 7. Pruebas

- `svc-datos-externos`:
  - cliente WFS con respuestas simuladas;
  - reproyección: un punto conocido en WGS84 cae en la parcela esperada;
  - deduplicación por nomenclatura entre puntos y lista, con su origen;
  - punto sin parcela y nomenclatura inexistente;
  - valor de mercado urbano (`vut × m²`) y rural (`vur × ha`);
  - mapa de valores caído: el valor fiscal se devuelve igual;
  - elección del año más reciente;
  - esquema con un campo faltante → 502.
- `svc-vivienda` (primer archivo de pruebas de `mi_lugar`):
  - el alta guarda aunque falle la consulta;
  - un error no pisa un resultado bueno;
  - `valor_fiscal` nunca cambia por la consulta;
  - reconciliación de nomenclaturas (alta, baja con soft delete, normalización, duplicadas);
  - el endpoint manual exige rol de escritura;
  - el endpoint `internal` recorre solo `exp` no borrados.
- Manual en el navegador: las tres filas de §3 muestran "= coincide", "= coincide" y "Δ −59%".
  Almafuerte coincide al cargar sus nomenclaturas faltantes.

## 8. Criterios de aceptación

1. En producción, Villa María y Estación Juárez Celman muestran "= coincide".
2. Ningún proceso automático cambió un `valor_fiscal` (verificar en el audit log).
3. Con `svc-datos-externos` apagado, el alta y la edición de un proyecto funcionan y las columnas
   dicen "no disponible".
4. Unidad del valor rural (§4.2) confirmada con IDECOR o con el área.
5. Cloud Scheduler semanal creado y con al menos una corrida `ok`.
6. Verificación visual del área en producción.

## 9. Preguntas abiertas

Ninguna. Las de la v0.1.0 se resolvieron en la v1.0.0 (ver changelog). La unidad del valor rural es
una verificación pendiente, no una decisión (§8, criterio 4).
