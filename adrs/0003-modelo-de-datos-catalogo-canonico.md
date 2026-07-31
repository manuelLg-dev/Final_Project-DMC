# ADR 0003: Modelo de datos con catálogo canónico curado

## Estado

Aceptado — enmendado el 2026-07-31 (ver "Enmienda 2026-07-31: `BusquedaRegistrada` normalizada y
vinculada").

## Contexto

Las pantallas del DESIGN.md revelan lo que el modelo debe soportar: comparación por tienda con
"mejor precio · Tienda" resaltado (2a), gráfico de historial de precios en el detalle (2b),
dashboard con "mayores caídas de precio, más buscados, índice de precios por categoría" (2c) y
catálogo por categorías (2d). El PRD exige además resolver su caso borde central: "El mismo
producto aparece con nombres/variantes ligeramente distintos entre tiendas... evitar comparar
productos que en realidad no son equivalentes", y advierte que el matching automático "merece su
propia investigación antes de diseño".

Durante esta decisión se detectó y resolvió un conflicto: el wireframe 2b incluye "alerta de
precio", pero el PRD la lista explícitamente en No alcance. Se decidió con el usuario retirarla
de v1 (queda como funcionalidad futura), por lo que el modelo no incluye entidad de alertas ni
datos de contacto de usuario.

## Decisión

El modelo se organiza alrededor de un catálogo canónico curado:

- **Categoria** — organiza el catálogo (pantallas 1c/2d) y soporta el índice de precios por
  categoría del dashboard.
- **Tienda** — las 4 tiendas soportadas (nombre, logo, URL base).
- **Producto** (canónico) — la entidad comparable: nombre normalizado, categoría, imagen,
  atributos distintivos (modelo/capacidad/color) para no mezclar variantes.
- **ListadoTienda** — la publicación de un producto canónico en una tienda concreta: URL del
  producto en la tienda y vínculo `Producto`–`Tienda`. El vínculo lo establece la semilla; el
  scraper solo alimenta listados ya mapeados. No hay matching automático en v1.
- **CapturaPrecio** — serie temporal por listado: precio, fecha/hora de captura, fuente
  (semilla | scraper) y flag de validez (para descartar precios absurdos, ver caso borde del PRD).
- **BusquedaRegistrada** — término buscado + fecha, necesaria porque el dashboard (2c) muestra
  "más buscados"; sin identificar al usuario (sin cuentas, per No alcance). Guarda el término
  crudo y su forma normalizada, y se vincula opcionalmente al `Producto` abierto desde esa
  búsqueda (ver enmienda del 2026-07-31).

La "mejor oferta vigente" y las tendencias del dashboard son datos derivados de `CapturaPrecio`,
no entidades propias.

## Alternativas consideradas

- **Listados independientes + matching automático (fuzzy)** — Refleja el problema real del
  dominio y escala a productos no curados, pero es exactamente el riesgo que el PRD pide
  investigar antes de diseñar; los falsos positivos producirían comparaciones inválidas, el peor
  error posible para la credibilidad del comparador. Inviable en plazo de curso.
- **Sin matching (cada producto pertenece a una tienda)** — Mínimo esfuerzo, pero rompe el
  objetivo central del PRD (comparar el mismo producto entre tiendas) y vacía de sentido la
  "mejor oferta vigente". Descartada por contradecir el alcance.

## Consecuencias

- La comparación entre tiendas es correcta por construcción: solo se comparan listados vinculados
  al mismo producto canónico, cumpliendo el caso borde de variantes del PRD sin algoritmo de
  matching.
- El historial y las tendencias salen de una única serie (`CapturaPrecio`), consultable tanto por
  producto (gráfico 2b) como agregada por categoría (dashboard 2c).
- **Trade-off:** el catálogo es cerrado — el scraper no puede descubrir productos nuevos, solo
  actualizar precios de listados ya curados. Escalar a catálogo abierto exigirá el matching
  automático que hoy se pospone (riesgo documentado en el PRD).
- Registrar búsquedas añade una escritura en el flujo de búsqueda que no existía en el PRD, pero
  es el costo de alimentar el "más buscados" que el wireframe del dashboard exige.

## Nota 2026-07-30: el esquema de `Producto` se amplía por el ADR-0009

El ADR-0009 (búsqueda full-text) modifica esta entidad: `Producto` gana una columna generada
`busqueda_tsv` de tipo `tsvector` con índice GIN, más un índice de trigramas sobre
`nombre_normalizado` para el fallback por similitud, y el sistema habilita las extensiones
`unaccent` y `pg_trgm`. Las demás entidades y las relaciones entre ellas no cambian.

Consecuencia para este ADR: `nombre_normalizado` deja de ser un campo de definición vaga y pasa a
tener un rol concreto — es el texto fuente de la indexación de búsqueda. Ver el ADR-0009 para el
detalle y el porqué.

## Enmienda 2026-07-31: `BusquedaRegistrada` normalizada y vinculada

**Qué cambió:** la entidad pasa de "término buscado + fecha" a tres campos, y la semilla la puebla:

| Campo | Rol |
|---|---|
| `termino_crudo` | Lo que el usuario escribió, tal cual. Sirve para calibrar el buscador con fallos reales (ADR-0009). |
| `termino_normalizado` | Clave de agregación del ranking: minúsculas, sin tildes (`unaccent`), espacios colapsados y lematizado con el diccionario `spanish` del ADR-0009. Se calcula en la ingesta del registro, no en la lectura. |
| `producto_id` (nullable) | El producto que el usuario abrió desde esa búsqueda, si abrió alguno. |

El ranking "más buscados" del dashboard **agrupa por `termino_normalizado`**. Cuando todos los
registros de un grupo apuntan al mismo `producto_id`, la tarjeta del wireframe 2c se renderiza con
los datos de ese producto (imagen, nombre, mejor precio); si no, se muestra como término de texto.

El generador de semilla (ADR-0001) **también siembra `BusquedaRegistrada`**: un volumen de
búsquedas distribuido en los últimos 7 días, con distribución desigual (unos términos claramente
más frecuentes que otros) y fechado relativo a `now()`, igual que las capturas.

**Por qué:** la revisión adversarial (`REVISION-ADVERSARIAL.md`, hallazgo A8) señaló tres cosas.
La primera, que con texto libre `laptop`, `Laptop`, `laptops` y `LAPTOP ` son cuatro filas
distintas y el ranking se fragmenta: el término más buscado podría no aparecer arriba por estar
repartido entre sus variantes, que es el fallo más silencioso posible en un dashboard. La segunda,
que el criterio de aceptación decía "términos/**productos**" y el wireframe 2c dibuja tarjetas de
producto, pero el modelo no tenía ningún vínculo con `Producto` que lo permitiera — la barra
inclinada escondía una decisión no tomada. La tercera, que el criterio de la semilla enumeraba
tiendas, categorías, productos, listados e historial pero **no búsquedas**: en una instalación
fresca la sección arrancaría vacía el día de la demo, salvo que alguien buscara a mano justo
antes.

Normalizar con el mismo diccionario del ADR-0009 no es casual: hace que "más buscados" agrupe con
el mismo criterio con el que el buscador encuentra, así que dos términos que devuelven el mismo
resultado cuentan como la misma búsqueda.

**Alternativas de la enmienda:**

- **Normalizar en la lectura, guardando solo el término crudo** — Permite cambiar la regla de
  normalización sin re-ingestar. Descartada por el mismo argumento que el ADR-0008 usó para
  validar en la ingesta: repartir la lógica por las consultas hace que un descuido en una sola
  produzca un ranking distinto. Guardar ambas formas conserva la posibilidad de recalcular.
- **Vincular la búsqueda al producto por match automático del término contra el catálogo** —
  Daría tarjeta de producto a más búsquedas que el clic real. Descartada porque reintroduce
  matching por texto —justo lo que este ADR evita— y porque un término ambiguo ("laptop") no
  identifica ningún producto concreto: el clic del usuario es la única señal fiable de qué
  producto buscaba.

**Consecuencias de la enmienda:**

- El ranking cuenta intenciones, no formas de escribir, y el dashboard puede mostrar las tarjetas
  de producto que el wireframe 2c dibuja.
- La sección "más buscados" tiene datos desde el primer arranque, coherente con la promesa del
  ADR-0001 de que la semilla garantiza una demo completa.
- **Trade-off:** los términos sembrados son inventados, igual que los precios. Quedan sujetos a la
  misma exigencia de honestidad que el hallazgo A7 plantea para las capturas de semilla.
- **Trade-off:** `producto_id` solo se conoce **después** de que el usuario hace clic, o sea que
  el registro se crea en el submit (ADR-0004, enmienda A1) y se actualiza al abrir el detalle. Es
  una escritura más en un flujo que el PRD no contemplaba; si resulta molesta, el ranking degrada
  limpiamente a términos de texto sin tarjetas, que sigue cumpliendo el criterio del PRD.
