# ADR 0009: Búsqueda con full-text de PostgreSQL (`tsvector` español + `unaccent`)

## Estado

Aceptado (2026-07-30)

## Contexto

El PRD fija como criterio de éxito medible: "El buscador central devuelve resultados relevantes
para al menos el 90% de búsquedas de productos que sí existen en el catálogo (medido con un set de
búsquedas de prueba)". `TECH-DESIGN.md` lo recoge como criterio de aceptación sobre un set de al
menos 20 búsquedas. El buscador es además la pantalla principal del producto (wireframe 1a) y la
entrada del flujo de 3 clics.

La revisión adversarial (`REVISION-ADVERSARIAL.md`, hallazgo C3) detectó que **ningún ADR decidía
cómo se busca**: el ADR-0006 justifica PostgreSQL solo por las agregaciones temporales del
dashboard y no menciona búsqueda; el ADR-0003 dice "nombre normalizado" sin definir qué normaliza.
El criterio más objetivamente medible del proyecto se estaba dejando a la implementación.

Restricciones y hechos del dominio que condicionan la decisión:

- **El catálogo es curado y pequeño** (ADR-0003: sin matching automático, listados vinculados a
  mano). Decenas o pocos cientos de productos, no miles. El rendimiento no es el problema; la
  **relevancia** sí.
- **Tildes:** el usuario peruano escribe indistintamente "television" y "televisión", "audifonos"
  y "audífonos". Una búsqueda sensible a acentos falla en la mitad de los casos reales.
- **Orden de palabras distinto:** "ideapad lenovo" debe encontrar "Lenovo IdeaPad 3 15.6 8GB".
- **Plurales y forma de la palabra:** "laptops" debe encontrar "Laptop"; "refrigeradoras" debe
  encontrar "Refrigeradora".
- **Coincidencia parcial del nombre:** casi ninguna búsqueda real repite el nombre canónico
  completo, que incluye atributos distintivos (modelo/capacidad/color, ADR-0003).
- Ya hay PostgreSQL en el sistema (ADR-0006) y un solo lenguaje/deploy (ADR-0005): añadir un motor
  de búsqueda aparte sería una tercera pieza de infraestructura para una persona.

## Decisión

La búsqueda se implementa con el **full-text search nativo de PostgreSQL**, con configuración de
diccionario `spanish` y la extensión `unaccent`:

- Se añade a `Producto` una columna `busqueda_tsv` de tipo `tsvector`, **generada** (columna
  generada almacenada) a partir de la concatenación de los campos buscables:
  `nombre_normalizado`, `marca`/`modelo` y demás atributos distintivos, más el nombre de la
  categoría si se decide incluirlo. La expresión aplica `unaccent()` antes de `to_tsvector('spanish', …)`.
- La consulta de búsqueda transforma el término del usuario con el mismo par
  `unaccent` + `websearch_to_tsquery('spanish', …)` y filtra con el operador `@@`.
- El orden de resultados usa `ts_rank`, con la posibilidad de ponderar los campos
  (`setweight`: nombre con peso `A`, atributos con peso `B`) si el set de prueba lo justifica.
- Se crea un índice **GIN** sobre `busqueda_tsv`.
- **Red de seguridad para errores de tipeo:** si `websearch_to_tsquery` no devuelve resultados, la
  consulta reintenta con similitud por trigramas (`pg_trgm`, `similarity()` sobre el nombre con un
  umbral) antes de mostrar el estado vacío. Es un segundo intento, no el camino principal.

Las extensiones requeridas (`unaccent`, `pg_trgm`) se habilitan en la migración inicial.

## Alternativas consideradas

- **`ILIKE '%termino%'` simple** — Cero configuración, cero extensiones, y con un catálogo de este
  tamaño el rendimiento sería perfectamente aceptable. Pero falla en tres de los cuatro casos
  reales del dominio: es sensible a tildes (`'%television%'` no encuentra "Televisión"), no maneja
  plurales ni raíces (`'%laptops%'` no encuentra "Laptop"), y exige que el término aparezca como
  subcadena **contigua**, así que "ideapad lenovo" no encuentra "Lenovo IdeaPad". Partir el
  término en palabras y encadenar `AND ILIKE` arregla lo último pero deja las tildes y los
  plurales, y equivale a reimplementar a mano y peor lo que el diccionario `spanish` ya hace.
  Descartada porque pone en riesgo el único criterio numérico del PRD para ahorrar una migración.
- **`pg_trgm` con similitud como mecanismo principal** — Muy tolerante a errores de tipeo y a
  variaciones de escritura, e insensible al orden si se compara por similitud de conjuntos de
  trigramas; sería la mejor opción si el problema dominante fueran las palabras mal escritas. Pero
  no entiende el idioma: no relaciona "laptops" con "laptop" salvo por parecido de caracteres, y
  su umbral de similitud es un valor difícil de calibrar que produce falsos positivos entre
  modelos parecidos ("IdeaPad 3" vs "IdeaPad 5") — precisamente el tipo de confusión entre
  variantes que el ADR-0003 puso tanto cuidado en evitar. Se adopta como **fallback**, no como
  mecanismo principal.
- **Motor de búsqueda externo (Meilisearch/Typesense/Elastic)** — Resolvería relevancia, tildes y
  tolerancia a errores de fábrica y con mejor experiencia de tuning. Pero añade un servicio más
  que instalar, desplegar y mantener sincronizado con PostgreSQL, para un catálogo de decenas de
  productos y un equipo de una persona: sobredimensionado, y agrava el problema de despliegue ya
  abierto en el hallazgo A4. Descartada por desproporción con la escala real.

## Impacto en el esquema del ADR-0003

Esta decisión modifica el modelo de datos definido en el ADR-0003. Cambios en la entidad
`Producto`:

| Cambio | Detalle |
|---|---|
| Columna nueva | `busqueda_tsv tsvector` — columna generada almacenada a partir de los campos buscables, con `unaccent` + `to_tsvector('spanish', …)` |
| Índice nuevo | Índice **GIN** sobre `busqueda_tsv` |
| Índice nuevo (fallback) | Índice **GIN/GiST** con `gin_trgm_ops` sobre `nombre_normalizado`, para la consulta de similitud |
| Extensiones | `unaccent` y `pg_trgm`, habilitadas en la migración inicial |

No cambian las demás entidades (`Categoria`, `Tienda`, `ListadoTienda`, `CapturaPrecio`,
`BusquedaRegistrada`) ni las relaciones entre ellas. `nombre_normalizado` queda ahora con una
definición concreta: es el texto fuente de la indexación, no un campo de propósito vago.

## Consecuencias

- El criterio del 90% pasa a ser **verificable y ajustable**: el set de al menos 20 búsquedas de
  prueba se convierte en la herramienta de calibración (pesos de `setweight`, campos incluidos en
  el `tsvector`, umbral de similitud del fallback), y se puede medir antes de la entrega en vez de
  descubrirse en la demo.
- No entra ninguna infraestructura nueva: todo vive en el PostgreSQL que el ADR-0006 ya justificó.
- Las tildes, los plurales y el orden de palabras quedan resueltos por el diccionario `spanish`,
  que es exactamente el problema que el `ILIKE` no podía cubrir.
- **Trade-off:** el diccionario `spanish` está pensado para lenguaje natural, no para catálogos de
  producto. Los tokens que más discriminan aquí son alfanuméricos ("15.6", "8GB", "IdeaPad 3") y
  el stemmer no los ayuda; peor, puede partirlos de forma poco intuitiva. Habrá que revisar contra
  el set de prueba qué pasa con modelos y capacidades, y posiblemente indexarlos en un campo
  aparte con su propio peso.
- **Trade-off:** una columna generada más un índice GIN encarece cada escritura de `Producto`.
  Irrelevante a esta escala (el catálogo se escribe por curación, no por ingesta continua: el
  scraper solo toca `CapturaPrecio`), pero conviene tenerlo presente si algún día se abre el
  catálogo.
- **Trade-off:** el fallback por trigramas introduce un segundo camino de código con su propio
  criterio de relevancia y su propio umbral. Es una regla más que mantener y calibrar; se acepta
  porque el estado vacío del PRD ("no un buscador roto") se paga caro con un usuario que
  simplemente escribió mal una letra.
- El registro de búsquedas (`BusquedaRegistrada`, ADR-0003) guarda el término **crudo** que
  escribió el usuario, no el `tsquery` normalizado: sirve para calibrar el buscador con búsquedas
  reales fallidas. La normalización de ese término para el ranking "más buscados" sigue sin
  decidir (hallazgo A8, abierto).
