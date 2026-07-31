# Revisión adversarial — AhorraPE

**Fecha:** 2026-07-30
**Documentos revisados:** `TECH-DESIGN.md`, `adrs/0001` a `adrs/0008`, `PRD.md`, `DESIGN.md`
**Condiciones de la revisión:** ejecutada en una conversación limpia, sin el historial de cómo se
generó el diseño. La revisión no arrastra el sesgo de defender lo propio.

**Naturaleza de este documento:** es un reporte de hallazgos. No se editó `TECH-DESIGN.md` ni
ningún ADR. Qué se cambia y qué se acepta como riesgo asumido lo decide el autor del diseño.

---

## Índice de hallazgos

| # | Severidad | Estado | Objetivo | Hallazgo |
|---|---|---|---|---|
| C1 | Crítico | ✅ Resuelto | ADR-0001 × ADR-0008 | La semilla envejece y la regla de 24 h apaga la demo |
| C2 | Crítico | ✅ Resuelto | ADR-0004 × ADR-0007 | Contradicción directa: el 0007 eligió lo que el 0004 rechazó |
| C3 | Crítico | ✅ Resuelto | Área faltante | Sin decisión sobre implementación de búsqueda (criterio del 90%) |
| A1 | Advertencia | ✅ Resuelto | ADR-0004 / ADR-0003 | Escritura dentro de un `GET`: "más buscados" corrompido |
| A2 | Advertencia | ✅ Resuelto | ADR-0007 | El caché por defecto de Next.js contradice "datos frescos" |
| A3 | Advertencia | ✅ Resuelto | ADR-0008 | Hueco de arranque de la validación y validación circular de la semilla |
| A4 | Advertencia | ✅ Resuelto | Área faltante | Sin decisión de dónde y cómo corren los jobs de ingesta |
| A5 | Advertencia | ✅ Resuelto | TECH-DESIGN | "Índice de precios por categoría" sin fórmula definida |
| A6 | Advertencia | ✅ Resuelto | TECH-DESIGN | Estado no definido: todos los listados desactualizados |
| A7 | Advertencia | ✅ Resuelto | ADR-0001 / ADR-0003 | Precios inventados atribuidos a tiendas reales con logo |
| A8 | Advertencia | ✅ Resuelto | ADR-0003 | `BusquedaRegistrada` sin normalización ni vínculo a producto |
| S1 | Sugerencia | ✅ Resuelto | Área faltante | Sin decisión de migraciones / capa de acceso a datos |
| S2 | Sugerencia | ✅ Resuelto | TECH-DESIGN | "Mayores caídas": ambigüedad producto vs listado |
| S3 | Sugerencia | ⚪ Aceptado como riesgo | ADR-0006 | Descarte de MySQL flojo |

### Resolución de los hallazgos críticos (2026-07-30)

| # | Resuelto en | Qué se decidió |
|---|---|---|
| C1 | [`adrs/0001`](adrs/0001-obtencion-de-precios-hibrida.md) — enmienda "anclaje temporal de la semilla" | El generador siembra timestamps relativos a `now()` en cada ejecución (última captura de cada listado dentro de las 24 h) y corre como **comando aparte**, no en el cron de 12 h. Criterios de aceptación añadidos en `TECH-DESIGN.md`. |
| C2 | [`adrs/0007`](adrs/0007-estado-en-servidor-y-url.md) — enmienda "las páginas consumen la API del ADR-0004"; nota en [`adrs/0004`](adrs/0004-contratos-api-rest-json.md) | Las páginas Server Component consumen los endpoints REST por `fetch` server-side. La API es el **único** camino de datos, no una superficie paralela para evaluación. Diagrama y descripción de componentes actualizados en `TECH-DESIGN.md`. |
| C3 | [`adrs/0009`](adrs/0009-busqueda-full-text-postgresql.md) — ADR nuevo; nota de impacto en [`adrs/0003`](adrs/0003-modelo-de-datos-catalogo-canonico.md) | Full-text de PostgreSQL con diccionario `spanish` + `unaccent` y ranking por `ts_rank`, con fallback por trigramas (`pg_trgm`) ante cero resultados. Descartados `ILIKE` simple y trigramas como mecanismo principal. Amplía `Producto` con `busqueda_tsv` + índices GIN. |

### Resolución de advertencias y sugerencias (2026-07-31)

| # | Resuelto en | Qué se decidió |
|---|---|---|
| A1 | [`adrs/0004`](adrs/0004-contratos-api-rest-json.md) — enmienda "el registro de búsqueda sale del `GET`" | `GET /api/busqueda` queda sin efectos secundarios; el registro pasa a `POST /api/busquedas`, invocado `fire-and-forget` desde el **submit del cliente**. Descartadas la deduplicación server-side (exigiría identificar al visitante, que el PRD excluye, y no arregla el prefetch) y la Server Action (contradiría la decisión C2). |
| A2 | [`adrs/0007`](adrs/0007-estado-en-servidor-y-url.md) — enmienda "política de caché explícita por tipo de ruta" | Rutas con precio o frescura → `force-dynamic` + `no-store`; dashboard → `revalidate` 5 min; catálogo → 1 h. Regla única: *¿muestra precio o frescura? → dinámica*. Descartado el `revalidate` corto en el detalle porque serviría un timestamp de frescura que no corresponde al estado real. |
| A3 | [`adrs/0008`](adrs/0008-validacion-en-ingesta-y-frescura-visible.md) — enmienda "primera captura de un listado" | La validación se desdobla: estructural (precio > 0 + rango absoluto de plausibilidad) siempre; desviación ±80% **solo si existe captura válida previa**. La referencia es la anterior en orden cronológico. Descartada la confirmación en dos lecturas por desproporción. |
| A4 | [`adrs/0010`](adrs/0010-ejecucion-de-jobs-de-ingesta.md) — ADR nuevo | **GitHub Actions con `schedule`** + `workflow_dispatch` como respaldo. El ADR es explícito en qué parte del "cada 12 h" es real: Actions encola los cron y desactiva workflows tras 60 días de inactividad, así que el criterio pasa a "corre automáticamente ~2 veces al día, sin intervención manual". Descartadas la ejecución manual (no demuestra ingesta continua, que es lo que el ADR-0001 quiere probar), el contenedor propio (desproporcionado) y Vercel Cron (sin navegador para Playwright). |
| A5 | `TECH-DESIGN.md` — sección "Fórmulas del dashboard" | **Canasta fija**: mismos productos con dato válido en `t` y `t−7d`, media de variaciones relativas, con el tamaño de la canasta visible y umbral de "datos insuficientes" bajo 3 productos. Evita que el índice se mueva por composición del catálogo. |
| A6 | `TECH-DESIGN.md` — criterios de comparación | Definido el caso "todos los listados desactualizados": sin fila resaltada ni tag "Mejor precio", aviso a nivel de producto con la fecha del dato más reciente, cada fila con su badge y su enlace, y el gráfico de historial se mantiene. |
| A7 | [`adrs/0001`](adrs/0001-obtencion-de-precios-hibrida.md) — enmienda "procedencia visible del dato" | `fuente` se expone en la API y se refleja en la UI: badge **"dato simulado"** por fila, más aviso de página cuando todo es semilla. Descartados los nombres genéricos de tienda (rompen el realismo y hacen incomparables las filas de semilla con las del scraper) y el disclaimer del footer como única señal. |
| A8 | [`adrs/0003`](adrs/0003-modelo-de-datos-catalogo-canonico.md) — enmienda "`BusquedaRegistrada` normalizada y vinculada" | Tres campos: `termino_crudo`, `termino_normalizado` (misma normalización del ADR-0009) y `producto_id` opcional por clic. El ranking agrupa por término normalizado y renderiza tarjeta de producto cuando el grupo apunta a uno solo. La semilla **también siembra búsquedas**, para que la sección no arranque vacía. |
| S1 | [`adrs/0011`](adrs/0011-acceso-a-datos-y-migraciones-drizzle.md) — ADR nuevo | **Drizzle ORM + Drizzle Kit**: esquema declarado en TypeScript como fuente única, migraciones SQL versionadas, y SQL crudo tipado para las agregaciones del dashboard y el full-text. Prisma descartado porque su esquema no expresa la columna generada `tsvector` ni los índices GIN del ADR-0009 y dejaría las agregaciones en `$queryRaw` sin tipar; SQL puro descartado porque no resuelve los "tipos compartibles" que motivaron el ADR-0005. |
| S2 | `TECH-DESIGN.md` — fórmulas y criterios del dashboard | "Mayores caídas" se calcula **por `ListadoTienda`**, no por producto: agregar por producto mezclaría series de tiendas distintas y produciría caídas que nadie puede aprovechar. |

### S3 — Aceptado como riesgo, sin cambio

El descarte de MySQL en el `ADR-0006` es flojo como argumentación ("sin ventaja técnica... solo se
justificaría por familiaridad previa"), pero **la observación es sobre la redacción, no sobre la
decisión**: PostgreSQL sigue siendo la elección correcta y el descarte fuerte —SQLite— está bien
argumentado. Reescribir el párrafo no cambiaría nada del sistema ni de ninguna otra decisión. Se
deja como está y se registra aquí para que conste que fue una decisión consciente y no un olvido.

---

## Crítico

### C1 — Los datos semilla envejecen y la regla de 24 h apaga la demo entera

> ✅ **Resuelto el 2026-07-30** en `adrs/0001-obtencion-de-precios-hibrida.md` (enmienda "anclaje
> temporal de la semilla"): timestamps relativos a `now()` en cada corrida, generador como comando
> aparte fuera del cron. Criterios de aceptación añadidos en `TECH-DESIGN.md`.

**Objetivo:** ADR-0001 en conflicto con ADR-0008.

`ADR-0001` promete que "la demo del curso nunca depende de un servicio externo: los datos semilla
garantizan catálogo, comparación e historial completos desde el día uno". `ADR-0008` decide que
toda captura con más de 24 h se marca "precio desactualizado" y **queda excluida del cálculo de
mejor oferta**.

Estas dos decisiones se anulan mutuamente. La semilla se genera con un comando ("de forma
repetible (un comando)", criterio de ingesta). Si se siembra el lunes y se presenta el jueves,
**todas** las capturas son de hace 72 h: cada fila del detalle muestra badge de desactualizado,
ningún listado califica para "mejor oferta vigente", y el resaltado amarillo del wireframe 2a —
el elemento visual central del producto — no aparece en ninguna pantalla. El dashboard de
"mayores caídas" sí sobrevive (usa 7 días), lo que hace el fallo más confuso: parte de la app
funciona.

Ningún ADR dice si el generador de semilla también corre en el ciclo de 12 h. El TECH-DESIGN dice
"Jobs programados cada 12 horas por tienda" sobre el pipeline en general, sin distinguir generador
de scraper.

**Por qué importa:** es la diferencia entre que la demo funcione o no. No es un detalle de
implementación.

**Dirección de arreglo:** decidir explícitamente que la semilla se regenera/refresca en el mismo
cron, o que genera capturas con timestamp relativo a `now()` en cada arranque.

---

### C2 — ADR-0004 y ADR-0007 se contradicen: el ADR-0007 eligió la alternativa que el ADR-0004 rechazó

> ✅ **Resuelto el 2026-07-30** en `adrs/0007-estado-en-servidor-y-url.md` (enmienda "las páginas
> consumen la API del ADR-0004"), con nota cruzada en `adrs/0004-contratos-api-rest-json.md`: las
> páginas Server Component consumen los endpoints por `fetch` server-side; la API es el único
> camino de datos. Diagrama de componentes actualizado en `TECH-DESIGN.md`.

**Objetivo:** ADR-0004 en conflicto con ADR-0007.

`ADR-0004` decide una API REST y **descarta explícitamente** "renderizado en servidor sin API
formal". `ADR-0007` decide exactamente eso: "Las pantallas se renderizan en el servidor con los
datos ya cargados (Server Components / SSR consultando la capa de datos del ADR-0004)".

Un Server Component no hace HTTP a `/api/*`; consulta la base directamente. Quedan dos lecturas y
el diseño no elige ninguna:

- Si las pantallas consultan la capa de datos directamente, los endpoints `/api/*` **no tienen
  consumidor**. Son código muerto que duplica la lógica de las páginas, y la consecuencia
  declarada del ADR-0004 ("el contrato es explícito y verificable con curl/Postman... facilita
  demostrar el backend en la evaluación") se vuelve el único motivo real de su existencia. Puede
  ser una razón válida en un proyecto de curso, pero entonces el ADR debe decir "la API existe
  para ser evaluable, no como camino de datos de producción", no presentarse como el contrato del
  frontend.
- Si el frontend sí llama a la API, entonces el trade-off del ADR-0004 ("estados de carga/error en
  el cliente") aplica y el ADR-0007 es falso.

**Síntoma de que nadie resolvió esto:** ningún criterio de aceptación menciona un solo endpoint.
Los criterios están todos escritos en términos de pantallas y URLs. La API no está siendo
verificada por nada.

**Dirección de arreglo:** una frase que decida si la API es camino de datos o superficie de
evaluación, y ajustar el ADR que quede desalineado.

---

### C3 — El criterio más medible del PRD no tiene ninguna decisión que lo sostenga

> ✅ **Resuelto el 2026-07-30** con el ADR nuevo
> `adrs/0009-busqueda-full-text-postgresql.md`: full-text de PostgreSQL (`spanish` + `unaccent`,
> ranking `ts_rank`) con fallback por trigramas; `ILIKE` simple y `pg_trgm` como mecanismo
> principal quedan descartados con su razón. Impacto en el esquema anotado en
> `adrs/0003-modelo-de-datos-catalogo-canonico.md` y en `TECH-DESIGN.md`.

**Objetivo:** área de decisión faltante (búsqueda).

El PRD fija: "El buscador central devuelve resultados relevantes para al menos el 90% de búsquedas
de productos que sí existen en el catálogo". El TECH-DESIGN lo copia como criterio de aceptación
(≥90% sobre ≥20 búsquedas). **No hay ningún ADR sobre cómo se busca.**

`ADR-0006` justifica PostgreSQL exclusivamente por las agregaciones temporales del dashboard; no
menciona búsqueda. `ADR-0003` dice "nombre normalizado" sin definir qué normaliza. La decisión por
defecto — `ILIKE '%q%'` — falla justo en los casos reales del dominio: tildes ("television" vs
"televisión"), errores de tipeo, orden de palabras ("ideapad lenovo" vs "Lenovo IdeaPad"),
plurales, marca sin modelo.

Con un catálogo curado y pequeño, 90% es alcanzable, pero requiere elegir: `tsvector` con
configuración `spanish` + `unaccent`, o `pg_trgm` con similitud, o un campo de sinónimos curado a
mano. Cada opción tiene costo distinto y afecta el esquema (columnas e índices), o sea que afecta
a `ADR-0003`.

**Por qué importa:** es el único número que el curso puede medir objetivamente y es el único que
el diseño no decidió.

**Dirección de arreglo:** falta un ADR.

---

## Advertencia

### A1 — Registrar la búsqueda dentro de un `GET` corrompe el "más buscados"

> ✅ **Resuelto el 2026-07-31** en `adrs/0004-contratos-api-rest-json.md` (enmienda "el registro de
> búsqueda sale del `GET`"): `POST /api/busquedas` desde el submit del cliente; el `GET` queda
> idempotente. Criterio de aceptación añadido: recargar o volver atrás no incrementa el conteo.

**Objetivo:** ADR-0004 (endpoint de búsqueda), ADR-0003 (`BusquedaRegistrada`).

`ADR-0004`: `GET /api/busqueda?q={termino}` — "resultados de búsqueda (y registra la búsqueda)".
Un `GET` con efecto secundario, combinado con las decisiones del `ADR-0007` (URL compartible,
recargable, botón atrás funcional), produce un contador inflado por construcción: cada recarga
(F5), cada vuelta con el botón atrás, cada apertura de un enlace compartido, cada crawler y cada
**prefetch de `<Link>` de Next.js** (activo por defecto) suma un registro.

El criterio de aceptación dice "Cada búsqueda **enviada** queda registrada" — pero bajo SSR no
existe un evento "enviada" distinguible de "página renderizada".

**Por qué importa:** el dashboard de tendencias es una de las cinco pantallas del producto y su
métrica sería ruido de navegación.

**Dirección de arreglo:** decidir dónde se registra (acción de submit explícita, no render) y con
qué deduplicación mínima.

---

### A2 — "Cada carga lee datos frescos del servidor" no es cierto en Next.js sin trabajo explícito

> ✅ **Resuelto el 2026-07-31** en `adrs/0007-estado-en-servidor-y-url.md` (enmienda "política de
> caché explícita por tipo de ruta"): `force-dynamic` + `no-store` en todo lo que muestre precio o
> frescura, `revalidate` en dashboard y catálogo. Criterio de aceptación añadido: recargar tras
> una corrida de ingesta refleja el precio nuevo.

**Objetivo:** ADR-0007 (consecuencias).

Consecuencia declarada del `ADR-0007`: "no hay caché de cliente ni store que invalidar cuando la
ingesta actualiza precios — cada carga lee datos frescos del servidor". El App Router de Next.js
hace lo contrario por defecto: renderiza estáticamente lo que puede y cachea rutas y `fetch`. Una
página de producto sin `revalidate`/`dynamic` explícito puede servir el precio del momento del
build.

**Por qué importa:** es exactamente el fallo que el `ADR-0008` intenta prevenir — mostrar un dato
obsoleto como vigente — reintroducido por el framework elegido en el `ADR-0005`. El ADR-0007 lo
presenta como una propiedad gratuita cuando es una configuración que hay que hacer y verificar.

**Dirección de arreglo:** añadir el costo a las consecuencias del ADR-0007 y un criterio de
aceptación: una recarga tras una corrida de ingesta debe reflejar el precio nuevo.

---

### A3 — La validación tiene un hueco de arranque y valida la semilla contra sí misma

> ✅ **Resuelto el 2026-07-31** en `adrs/0008-validacion-en-ingesta-y-frescura-visible.md`
> (enmienda "primera captura de un listado"): validación estructural siempre (precio > 0 + rango
> absoluto), desviación ±80% solo con captura previa, y referencia en orden cronológico. Riesgo
> residual anotado en `TECH-DESIGN.md` ("ancla falsa en la primera captura").

**Objetivo:** ADR-0008.

`ADR-0008` valida "desviación acotada frente a **la última captura válida del mismo listado**".
Dos problemas que el ADR no toca:

- **Primera captura de un listado:** no hay captura previa. ¿Se acepta cualquier precio > 0?
  Entonces un scraper que arranca leyendo mal el HTML establece la línea base y la regla ±80%
  queda anclada a un valor falso — y la siguiente captura, la correcta, se rechaza por desviarse
  de él. La regla es más frágil al inicio que en régimen, que es justo cuando el scraper es menos
  confiable.
- **Orden de inserción de la semilla:** sembrar 4 semanas de historial en una corrida exige
  procesar cronológicamente. Si el generador inserta fuera de orden, "la última captura válida" es
  un punto arbitrario de la serie y capturas perfectamente buenas se marcan inválidas. Nadie
  especificó que la ingesta valide en orden temporal.

**Trade-off no reconocido:** hacer pasar la semilla por la misma validación no aporta ninguna
garantía (el generador produce datos que él mismo controla) pero sí crea un modo de fallo nuevo —
que los datos de la demo salgan marcados como inválidos. El `ADR-0001` tiene una buena razón para
el pipeline compartido (que el scraper demuestre algo sobre la plataforma real), pero el costo
específico en la semilla no está anotado en ningún lado.

---

### A4 — Nadie decidió dónde corren los jobs de ingesta

> ✅ **Resuelto el 2026-07-31** con el ADR nuevo `adrs/0010-ejecucion-de-jobs-de-ingesta.md`:
> GitHub Actions programado, con `workflow_dispatch` como respaldo. El ADR es explícito en qué
> parte del "cada 12 h" es real y qué no; el criterio de aceptación se reformuló como "corre
> automáticamente ~2 veces al día, sin intervención manual", verificable con el historial de
> ejecuciones del workflow.

**Objetivo:** área de decisión faltante (despliegue/scheduling); afecta a ADR-0002, ADR-0005,
ADR-0006.

`ADR-0002` apoya buena parte de su valor en que "la ingesta corre y falla de forma aislada".
`ADR-0005` elige Playwright. `ADR-0006` propone Supabase o Neon "si el proyecto se despliega".
Falta la pieza que las une: **qué ejecuta el cron de 12 h**.

Si la webapp va a Vercel, no puede alojar un proceso Playwright de larga duración; las Vercel Cron
Functions tienen límites de tiempo y no traen navegador. Las opciones reales (GitHub Actions con
schedule, un contenedor en Fly/Railway, o simplemente "corre en mi laptop y para la demo lo
ejecuto a mano") tienen implicaciones muy distintas.

**Por qué importa:** el criterio "Los jobs de ingesta corren cada 12 h" es hoy **inverificable**.
Si la respuesta honesta es "manual para la demo", hay que escribirlo — y entonces C1 se agrava.

---

### A5 — El "índice de precios por categoría" no está definido en ninguna parte

> ✅ **Resuelto el 2026-07-31** en `TECH-DESIGN.md`, nueva sección "Fórmulas del dashboard":
> canasta fija (mismos productos en `t` y `t−7d`, media de variaciones relativas), con tamaño de
> canasta visible y "datos insuficientes" bajo 3 productos.

**Objetivo:** TECH-DESIGN, criterios del dashboard; ningún ADR lo cubre.

Es una de las tres secciones del dashboard (2c) y el criterio de aceptación solo dice que se
calcula "con capturas válidas" y muestra "la variación de los últimos 7 días". No hay fórmula.

Un promedio simple de precios de la categoría se mueve por **composición**, no por precio: si un
producto caro entra al catálogo, o si un producto barato queda excluido por estar desactualizado
(regla del ADR-0008), el índice salta sin que ningún precio haya cambiado.

**Por qué importa:** mostrar "Tecnología +14%" cuando nada subió es un error silencioso en un
producto cuya única propuesta de valor es que sus números de precio son confiables.

**Dirección de arreglo:** un índice tipo canasta fija (mismo conjunto de productos en t y t−7d,
media de variaciones relativas) lo evita — pero es una decisión, y hay que tomarla.

---

### A6 — Estado no definido: todos los listados de un producto desactualizados

> ✅ **Resuelto el 2026-07-31** en los criterios de comparación de `TECH-DESIGN.md`: sin fila
> resaltada ni tag "Mejor precio", aviso a nivel de producto con la fecha del dato más reciente,
> cada fila conserva su badge y su enlace, y el gráfico de historial se mantiene.

**Objetivo:** TECH-DESIGN, criterios de comparación; DESIGN.md (pendientes).

Los criterios cubren "solo una tienda" y "un listado sin captura válida en 24 h" por separado. No
cubren la intersección, que con un scraper best-effort y semilla envejecida (C1) es probable:
**ningún** listado del producto tiene captura válida. ¿Qué muestra el detalle? ¿Precios todos con
badge y sin resaltado? ¿Un estado vacío? ¿El gráfico de historial sigue?

El `DESIGN.md` lista "estado precio desactualizado" como pendiente y esto es la variante peor del
mismo pendiente.

**Por qué importa:** sin definirlo, la implementación va a improvisar y probablemente renderice
una comparación sin ganador, que es justo el "no simular una comparación inexistente" del PRD.

---

### A7 — Precios inventados atribuidos a tiendas reales con nombre y logo

> ✅ **Resuelto el 2026-07-31** en `adrs/0001-obtencion-de-precios-hibrida.md` (enmienda
> "procedencia visible del dato"): badge **"dato simulado"** por fila cuando `fuente = semilla`,
> aviso de página cuando todo lo mostrado es semilla, y `fuente` expuesta en la API.

**Objetivo:** ADR-0001 (semilla) y ADR-0003 (entidad `Tienda`).

`ADR-0003` modela `Tienda` con "nombre, logo, URL base" y las 4 son empresas reales. `ADR-0001`
genera precios sintéticos. El resultado es una pantalla que dice "Falabella — S/ 2,499" con el
logo de Falabella, siendo el precio inventado.

El `ADR-0001` reconoce el riesgo de scraping/ToS y lo mitiga; este es un riesgo **distinto** que
ningún ADR menciona: atribuir un precio falso a un negocio identificado no es lo mismo que extraer
uno real sin permiso, y un disclaimer en el footer ("Precios referenciales") no cubre la
atribución nominal.

**Por qué importa:** para un entregable de curso el alcance del daño es mínimo, pero la mitigación
también es barata, y además hace honesta la demo: hoy el evaluador no puede distinguir un precio
scrapeado de uno generado.

**Dirección de arreglo:** marcar visiblemente en la fila las capturas con `fuente = semilla`, o
usar nombres genéricos ("Tienda A") en los datos sembrados.

---

### A8 — `BusquedaRegistrada` guarda texto libre; el dashboard promete "términos/productos"

> ✅ **Resuelto el 2026-07-31** en `adrs/0003-modelo-de-datos-catalogo-canonico.md` (enmienda
> "`BusquedaRegistrada` normalizada y vinculada"): término crudo + normalizado + `producto_id`
> opcional; el ranking agrupa por término normalizado; la semilla también siembra búsquedas.

**Objetivo:** ADR-0003, criterios del dashboard y de la semilla.

`ADR-0003` define la entidad como "término buscado + fecha", sin vínculo a `Producto` y sin
normalización. El criterio de aceptación dice "'Más buscados' lista los **términos/productos**" —
esa barra inclinada esconde una decisión no tomada. Con texto libre, `laptop`, `Laptop`, `laptops`
y `laptop hp` son cuatro filas distintas y el ranking se fragmenta; el wireframe 2c sugiere
tarjetas de producto, que exigirían un vínculo que el modelo no tiene.

Además, en una instalación fresca hay **cero** búsquedas: el criterio de la semilla enumera
tiendas, categorías, productos, listados e historial, pero no búsquedas. La sección "más buscados"
del dashboard arrancará en estado vacío el día de la demo, salvo que alguien busque a mano justo
antes.

---

## Sugerencia

### S1 — Sin decisión de migraciones / capa de acceso a datos

> ✅ **Resuelto el 2026-07-31** con el ADR nuevo
> `adrs/0011-acceso-a-datos-y-migraciones-drizzle.md`: Drizzle ORM + Drizzle Kit, esquema en
> TypeScript como fuente única, migraciones SQL versionadas, SQL crudo para las agregaciones.

`ADR-0002` establece que "la base de datos se convierte en el contrato entre componentes:
cualquier cambio de esquema afecta a ambos y debe versionarse con cuidado (migraciones)", y
`ADR-0005` apoya el argumento de "tipos compartibles" para mitigar la desincronización del
`ADR-0004`. Ninguno decide la herramienta (Prisma, Drizzle, SQL + node-pg-migrate). Es la pieza
que hace real ese "tipos compartibles" y afecta a los dos componentes; merece un ADR corto en vez
de resolverse en el primer commit.

### S2 — "Mayores caídas" es ambiguo y, con semilla, es ficción

> ✅ **Resuelto el 2026-07-31** en `TECH-DESIGN.md` (fórmulas y criterios del dashboard): se
> calcula por `ListadoTienda`, no por producto. La segunda parte (que con semilla el ranking es el
> random walk del generador) queda cubierta por la resolución de A7: las filas de semilla llevan
> indicador visible.

El criterio dice "productos ordenados por mayor descenso porcentual... indicando tienda",
mezclando nivel producto y nivel listado; conviene fijar que se calcula por `ListadoTienda`.
Aparte: en la demo, ese ranking es literalmente el random walk del generador. No es un defecto de
diseño, pero sí algo que conviene saber antes de que alguien en la presentación pregunte por qué
cayó ese producto.

### S3 — ADR-0006, descarte de MySQL

> ⚪ **Aceptado como riesgo el 2026-07-31, sin cambio.** La observación es sobre la redacción del
> ADR, no sobre la decisión: PostgreSQL sigue siendo correcto y el descarte fuerte (SQLite) está
> bien argumentado. Reescribir el párrafo no cambiaría nada del sistema.

"Sin ventaja técnica sobre PostgreSQL... solo se justificaría por familiaridad previa" es cierto
pero es la parte más floja del set de alternativas. El descarte real y fuerte era SQLite y está
bien argumentado; MySQL está ahí para llenar el hueco de "dos alternativas".

---

## Lo que aguantó el escrutinio

No todo tenía problemas. Estos ADRs fueron desafiados y no encontré con qué tumbarlos:

- **ADR-0002 (dos componentes)** — Sólido. El contexto justifica de verdad, ambas alternativas son
  opciones reales (no una falsa elección), el trade-off del deploy compartido es concreto y
  anticipa el "flujo mobile" del DESIGN.md, y la consecuencia de que la DB se vuelve contrato es
  la clase de costo que la mayoría de los ADRs omite. Su único hueco es A4, que es una decisión
  faltante, no un error del ADR.
- **ADR-0003 (catálogo canónico)** — Resuelve bien el caso borde central del PRD y su trade-off
  (catálogo cerrado, el scraper no descubre productos) es el costo real, no un adorno. Detectar y
  resolver el conflicto de la alerta de precio contra el No-alcance del PRD, y dejarlo escrito, es
  exactamente lo que debe hacer un ADR. Los problemas encontrados (A8, C3) son de cobertura, no de
  razonamiento.
- **ADR-0005 (stack TypeScript)** — Proporcionado a la escala del proyecto y su trade-off es el
  correcto y honesto (ecosistema de scraping más pobre en Node). Intenté argumentar que era
  sobreingeniería o subingeniería para un proyecto de una persona y no encontré por dónde.

---

## Resumen de prioridad

- ~~**C1** es el que rompe la entrega — se arregla decidiendo cómo se refrescan los timestamps de
  la semilla.~~ ✅ Resuelto (ADR-0001, enmienda).
- ~~**C2** exige elegir, en una frase, si la API es camino de datos o superficie de evaluación, y
  ajustar el ADR que quede desalineado.~~ ✅ Resuelto (ADR-0007, enmienda; nota en ADR-0004).
- ~~**C3** pide un ADR nuevo.~~ ✅ Resuelto (ADR-0009).
- ~~**A1–A8** siguen abiertos...~~ ✅ Resueltos el 2026-07-31 (ver la tabla de resolución arriba).
  Dos ADRs nuevos (0010 ejecución de jobs, 0011 acceso a datos), cinco ADRs enmendados (0001,
  0003, 0004, 0007, 0008) y `TECH-DESIGN.md` ampliado con la sección "Fórmulas del dashboard" y
  criterios de aceptación nuevos.
- **S1–S2** ✅ resueltos. **S3** aceptado como riesgo, sin cambio.

### Riesgos residuales tras la resolución

Ninguna de estas decisiones elimina su problema por completo. Lo que queda vivo está anotado en
"Riesgos técnicos abiertos" de `TECH-DESIGN.md`:

- El rango absoluto de plausibilidad (A3) reduce el ancla falsa pero no la elimina: un error que
  caiga dentro del rango sigue pudiendo fijarse como referencia.
- GitHub Actions (A4) no garantiza puntualidad y desactiva los cron tras 60 días de inactividad;
  además sale por IP de datacenter, lo que **aumenta** la probabilidad de que el scraper sea
  bloqueado — un agravante nuevo sobre el riesgo de viabilidad del scraping.
- `force-dynamic` en las rutas de precio (A2) elimina el caché justo donde más tráfico habría.
- El registro por submit (A1) subestima las búsquedas si el cliente no ejecuta JavaScript;
  se prefirió subestimar a inflar.

Ninguno de ellos es un descuido: los cuatro son el precio explícito de la decisión que los
produjo.

---

## Estado final

**14 hallazgos: 13 resueltos, 1 aceptado como riesgo.** El diseño pasó de 8 a 11 ADRs.

Conviene decir algo que este reporte no puede verificar por sí mismo: resolver un hallazgo aquí
significa que **existe una decisión escrita y justificada**, no que el sistema funcione. La
verificación real ocurre contra los criterios de aceptación cuando haya código — y varios de los
criterios añadidos (la canasta fija de A5, el caché de A2, el conteo de búsquedas de A1) están
escritos precisamente porque son fallos que no se ven en pantalla si nadie los mide.
