# Technical Design Document: AhorraPE

**Tipo de proyecto:** Greenfield
**Design.md disponible:** Sí (`DESIGN.md` — documentación de wireframes; el modelo de datos se
derivó del PRD y de lo que las pantallas revelan que debe mostrarse).

## Resumen

AhorraPE es un comparador de precios de tiendas online peruanas (PRD: `PRD.md`): el usuario busca
un producto desde un buscador central y ve en una sola pantalla su precio en las tiendas que lo
venden, cuál es la mejor oferta vigente y cómo ha evolucionado el precio en el tiempo. Esta
versión se dimensiona como **entregable de curso (DMC)** con 4 tiendas objetivo — Falabella,
Ripley, Plaza Vea y Oechsle — y una estrategia de datos híbrida: datos semilla que garantizan la
demo y un scraper best-effort que demuestra la viabilidad de la ingesta real (ADR-0001).

## Arquitectura de componentes

Dos componentes en un mismo repositorio, comunicados únicamente a través de la base de datos
(ADR-0002):

```
┌─────────────────────────────┐      ┌──────────────────────────────────┐
│  Pipeline de ingesta (jobs) │      │  Aplicación web (Next.js)        │
│  · Generador de semilla     │      │  ┌────────────────────────────┐  │
│    (comando aparte)         │      │  │ Páginas / Server Components│  │
│  · Scraper best-effort      │      │  └─────────────┬──────────────┘  │
│    (1-2 tiendas, ADR-0010)  │      │       fetch server-side          │
│  → módulo de ingesta común  │      │  ┌─────────────▼──────────────┐  │
│    con validación (ADR-0008)│      │  │  API REST /api/* (ADR-0004)│  │
└──────────────┬──────────────┘      │  └─────────────┬──────────────┘  │
               │ escribe             └────────────────┼─────────────────┘
               │                                      │ lee
               ▼                                      ▼
          ┌─────────────────────────────────────────────┐
          │            PostgreSQL (ADR-0006)            │
          └─────────────────────────────────────────────┘
```

- **Aplicación web** (Next.js, TypeScript — ADR-0005): sirve las cinco pantallas del DESIGN.md
  (home/buscador, resultados, detalle con historial, dashboard de tendencias, categorías) y
  expone la API REST interna. Las páginas se renderizan en servidor y obtienen sus datos
  **llamando a esa misma API por `fetch` server-side** (ADR-0004 + ADR-0007): la API es el único
  camino de datos, no una superficie paralela. La URL es el estado de interfaz (ADR-0007). La
  búsqueda usa full-text de PostgreSQL (ADR-0009).
- **Pipeline de ingesta** (scripts Node/TypeScript — ADR-0005): generador de semilla y scraper
  escriben por un módulo de ingesta común que valida cada captura (ADR-0008). El **scraper** corre
  como job programado en GitHub Actions, ~2 corridas diarias por tienda (ADR-0010); el **generador
  de semilla** es un comando aparte que se ejecuta antes de cada demo y siembra timestamps
  relativos a `now()` (ADR-0001, enmienda 2026-07-30). El esquema y las migraciones se gestionan
  con Drizzle (ADR-0011).

## Decisiones de arquitectura

| # | Decisión | Estado |
|---|---|---|
| [ADR-0001](adrs/0001-obtencion-de-precios-hibrida.md) | Obtención de precios híbrida (semilla + scraper best-effort) | Aceptado (enmendado 2026-07-30 y 2026-07-31) |
| [ADR-0002](adrs/0002-componentes-webapp-mas-ingesta.md) | Dos componentes: webapp full-stack + pipeline de ingesta | Aceptado |
| [ADR-0003](adrs/0003-modelo-de-datos-catalogo-canonico.md) | Modelo de datos con catálogo canónico curado | Aceptado (enmendado 2026-07-31) |
| [ADR-0004](adrs/0004-contratos-api-rest-json.md) | API REST JSON interna como contrato frontend–datos | Aceptado (enmendado 2026-07-31) |
| [ADR-0005](adrs/0005-stack-typescript-nextjs.md) | Stack TypeScript end-to-end (Next.js + scripts Node) | Aceptado |
| [ADR-0006](adrs/0006-base-de-datos-postgresql.md) | PostgreSQL como base de datos compartida | Aceptado |
| [ADR-0007](adrs/0007-estado-en-servidor-y-url.md) | Estado en servidor con la URL como estado de interfaz | Aceptado (enmendado 2026-07-30 y 2026-07-31) |
| [ADR-0008](adrs/0008-validacion-en-ingesta-y-frescura-visible.md) | Validación de precios en la ingesta y frescura visible | Aceptado (enmendado 2026-07-31) |
| [ADR-0009](adrs/0009-busqueda-full-text-postgresql.md) | Búsqueda con full-text de PostgreSQL (`tsvector` español + `unaccent`) | Aceptado |
| [ADR-0010](adrs/0010-ejecucion-de-jobs-de-ingesta.md) | Ejecución de los jobs de ingesta (GitHub Actions programado) | Aceptado |
| [ADR-0011](adrs/0011-acceso-a-datos-y-migraciones-drizzle.md) | Acceso a datos y migraciones con Drizzle ORM | Aceptado |

### Cambios tras la revisión adversarial

Reporte completo en [`REVISION-ADVERSARIAL.md`](REVISION-ADVERSARIAL.md).

**Hallazgos críticos (2026-07-30):**

- **C1** — El generador de semilla siembra timestamps **relativos a `now()`** en cada ejecución, y
  corre como comando aparte (no en el cron). Sin esto, la ventana de frescura de 24 h del
  ADR-0008 dejaba toda la semilla fuera del cálculo de "mejor oferta vigente" a los pocos días
  (ADR-0001, enmienda).
- **C2** — Las páginas Server Component **consumen la API REST del ADR-0004** por `fetch`
  server-side. La API es el único camino de datos; se descarta la lectura directa a la base desde
  las páginas, que dejaba los endpoints sin consumidor (ADR-0007, enmienda).
- **C3** — La búsqueda se decide en el nuevo **ADR-0009**: full-text de PostgreSQL con diccionario
  `spanish` + `unaccent`, y fallback por trigramas. Amplía el esquema de `Producto` del ADR-0003.

**Advertencias y sugerencias (2026-07-31):**

- **A1** — El registro de búsqueda sale del `GET` y pasa a `POST /api/busquedas`, invocado en el
  submit del cliente: recargas, botón atrás y prefetch de `<Link>` dejan de inflar "más buscados"
  (ADR-0004, enmienda).
- **A2** — Política de caché explícita por tipo de ruta: las que muestran precio o frescura son
  `force-dynamic`; dashboard y catálogo usan `revalidate` (ADR-0007, enmienda).
- **A3** — La regla de ±80% solo aplica cuando existe captura previa; la primera captura de un
  listado se valida contra un rango absoluto de plausibilidad (ADR-0008, enmienda).
- **A4** — Los jobs de ingesta corren en **GitHub Actions programado**, con ejecución manual como
  respaldo (nuevo **ADR-0010**, que precisa qué parte del "cada 12 h" es real).
- **A5** — El índice de precios por categoría se define como **canasta fija** (fórmula abajo).
- **A6** — Definido el caso "todos los listados desactualizados" en los criterios de aceptación.
- **A7** — La procedencia del dato (`semilla` | `scraper`) se hace **visible en la interfaz**, para
  no atribuir precios inventados a tiendas reales (ADR-0001, enmienda).
- **A8** — `BusquedaRegistrada` guarda término crudo + normalizado y se vincula opcionalmente a
  `Producto`; la semilla también siembra búsquedas (ADR-0003, enmienda).
- **S1** — **Drizzle ORM** + Drizzle Kit como capa de datos y migraciones (nuevo **ADR-0011**).
- **S2** — "Mayores caídas" se calcula por `ListadoTienda` (precisado en los criterios).
- **S3** — Aceptado como riesgo, sin cambio: la debilidad del descarte de MySQL en el ADR-0006 es
  de redacción, no de decisión; reescribirlo no cambiaría nada del sistema.

Decisiones de alcance tomadas durante el diseño:

- La "alerta de precio" del wireframe 2b **se retira de v1**: contradecía el No-alcance del PRD
  (alertas automáticas). Queda como funcionalidad futura (ADR-0003).
- Los picos de tráfico tipo CyberWow quedan **fuera del diseño de v1**, documentados como riesgo
  conocido (ADR-0008).
- Frecuencia de actualización de precios: **~2 corridas diarias** por tienda (resuelve el "REVISAR"
  del PRD sobre frecuencia). El ADR-0010 precisa el alcance real de esa promesa: GitHub Actions
  encola los cron y no garantiza puntualidad exacta.
- Ventanas del dashboard: "más buscados" y "mayores caídas" se calculan sobre los **últimos 7
  días**.

## Fórmulas del dashboard

### Índice de precios por categoría (canasta fija)

El índice **no es el promedio de precios de la categoría**. Se calcula como una canasta fija sobre
los productos que tienen dato válido en ambos extremos de la ventana:

1. Sea `C` el conjunto de productos de la categoría con al menos una captura válida en `t` (hoy) y
   otra en `t−7d`. Un producto sin dato en cualquiera de los dos extremos **queda fuera de `C`**.
2. Para cada producto `p ∈ C`, se toma su mejor precio vigente en cada extremo (mínimo entre sus
   listados, mismas reglas de validez del ADR-0008): `precio(p, t)` y `precio(p, t−7d)`.
3. La variación del producto es `v(p) = precio(p, t) / precio(p, t−7d) − 1`.
4. **El índice de la categoría es la media de `v(p)` sobre `C`**, expresada en porcentaje.
5. La UI muestra junto al índice **cuántos productos componen la canasta**; si `|C| < 3`, la
   categoría muestra "datos insuficientes" en vez de un porcentaje.

**Por qué no un promedio simple** (hallazgo A5): la media de precios de una categoría se mueve por
**composición**, no por precio. Si entra un producto caro al catálogo, o si uno barato queda
excluido por estar desactualizado (regla de 24 h del ADR-0008), el índice salta sin que ningún
precio haya cambiado. Mostrar "Tecnología +14%" cuando nada subió es un error silencioso en un
producto cuya única propuesta de valor es que sus números son confiables. La canasta fija compara
los mismos productos consigo mismos, así que solo se mueve cuando se mueven los precios.

**Trade-off asumido:** con pocos productos por categoría, la exclusión de los que no tienen dato en
ambos extremos puede dejar canastas muy pequeñas — de ahí el umbral de `|C| < 3` y el mostrar el
tamaño de la canasta, en vez de un porcentaje que aparente más solidez de la que tiene.

### Mayores caídas de precio

Se calcula **por `ListadoTienda`, no por `Producto`** (hallazgo S2): la unidad es "este producto en
esta tienda". El descenso porcentual se mide entre la primera y la última captura **válida** del
listado dentro de los últimos 7 días, y cada fila del ranking nombra producto **y** tienda.

Agregar por producto mezclaría series de tiendas distintas y produciría caídas que ningún usuario
puede aprovechar (el "mínimo entre tiendas" puede bajar solo porque una tienda distinta entró al
rango, sin que nadie haya rebajado nada).

## Modelo de datos

Entidades (ADR-0003):

- **Categoria** `1—N` **Producto** — catálogo navegable (pantallas 1c/2d) e índice de precios por
  categoría del dashboard.
- **Producto** (canónico: nombre normalizado, imagen, atributos distintivos como
  modelo/capacidad/color) `1—N` **ListadoTienda**. Incluye además la columna generada
  `busqueda_tsv` (`tsvector`) con índice GIN, e índice de trigramas sobre el nombre normalizado
  para el fallback de búsqueda (ADR-0009).
- **Tienda** (las 4 soportadas: nombre, logo, URL base) `1—N` **ListadoTienda**.
- **ListadoTienda** (URL del producto en la tienda; el vínculo con el producto canónico lo
  establece la curación/semilla — sin matching automático en v1) `1—N` **CapturaPrecio**.
- **CapturaPrecio** — serie temporal: precio, fecha/hora, fuente (`semilla` | `scraper`), flag de
  validez (ADR-0008). La `fuente` se expone en la API y se refleja en la UI (ADR-0001, enmienda
  2026-07-31).
- **BusquedaRegistrada** — término crudo + término normalizado + fecha/hora + `producto_id`
  opcional (el producto abierto desde esa búsqueda); sin identificar usuario. Alimenta el "más
  buscados" del dashboard, agrupando por término normalizado (ADR-0003, enmienda 2026-07-31).

El esquema se declara en TypeScript con Drizzle y se versiona con migraciones SQL en el
repositorio (ADR-0011).

Derivados (no son entidades): la **mejor oferta vigente** de un producto es el mínimo de las
capturas válidas de las últimas 24 h entre sus listados; las **tendencias** del dashboard se
agregan de `CapturaPrecio` (caídas en 7 días, índice por categoría) y `BusquedaRegistrada`
(más buscados en 7 días).

## Criterios de aceptación por flujo

### Búsqueda de producto

- [ ] Desde el home, escribir un término y enviar lleva a `/buscar?q={termino}` con resultados
      renderizados en servidor; la URL es compartible y recargable con el mismo resultado.
- [ ] Buscar un producto existente en el catálogo lo devuelve entre los resultados (objetivo del
      PRD: ≥90% de acierto sobre un set de prueba definido de al menos 20 búsquedas).
- [ ] El set de prueba incluye casos con tildes omitidas ("television"), plurales ("laptops"),
      orden de palabras invertido ("ideapad lenovo") y un error de tipeo, y todos devuelven el
      producto esperado (ADR-0009).
- [ ] Buscar un término sin coincidencias muestra el estado vacío diseñado (mensaje claro +
      sugerencia de categorías), nunca una lista vacía sin explicación ni un error.
- [ ] Cada búsqueda enviada queda registrada mediante `POST /api/busquedas` **desde el submit del
      buscador**, con término crudo, término normalizado y fecha/hora, sin datos que identifiquen
      al usuario (ADR-0004, enmienda A1).
- [ ] `GET /api/busqueda` no escribe nada: recargar la página de resultados, volver con el botón
      atrás o abrir un enlace compartido **no** incrementa el conteo de "más buscados".
- [ ] Abrir un producto desde los resultados vincula la búsqueda registrada con ese `producto_id`
      (ADR-0003, enmienda A8).
- [ ] El flujo home → resultados → detalle de producto toma como máximo 3 clics (criterio del
      PRD).

### Comparación de precios en el detalle de producto

- [ ] El detalle muestra una fila por tienda con listado del producto, cada una con precio
      vigente, fecha/hora de la última captura válida y enlace a la tienda original.
- [ ] La mejor oferta vigente es el precio mínimo entre capturas **válidas** de las **últimas
      24 h**, y se resalta con el tratamiento del wireframe (fondo/tag "Mejor precio · Tienda")
      sobre la fila completa.
- [ ] Si un listado no tiene captura válida en las últimas 24 h, su precio se muestra con badge
      de "precio desactualizado" con la fecha de última captura, y queda excluido del cálculo de
      mejor oferta.
- [ ] Si el producto tiene listado en una sola tienda, se muestra solo esa tienda con su precio,
      sin tag de "mejor precio" ni simulación de comparación (caso borde del PRD).
- [ ] **Si ningún listado del producto tiene captura válida en las últimas 24 h** (todos
      desactualizados, hallazgo A6): la página **no resalta ninguna fila** ni muestra el tag
      "Mejor precio"; encabeza un aviso a nivel de producto ("no hay precios vigentes; el dato más
      reciente es de {fecha}"); cada fila conserva su último precio válido con su badge de
      desactualizado y su fecha; el enlace a la tienda original sigue disponible; y el gráfico de
      historial se muestra normalmente, ya que su valor no depende de la ventana de 24 h.
- [ ] Una captura con precio 0, fuera del rango absoluto de plausibilidad, o —**cuando ya existe
      una captura válida previa del mismo listado**— con desviación mayor a ±80% respecto de ella,
      nunca aparece como precio vigente ni como mejor oferta (queda persistida con flag de
      inválida). La primera captura de un listado no se somete a la regla de desviación (ADR-0008,
      enmienda A3).
- [ ] Toda fila cuya última captura válida tenga `fuente = semilla` muestra el indicador visible
      de **"dato simulado"**; si todas las filas de la pantalla son de semilla, se muestra además
      el aviso a nivel de página (ADR-0001, enmienda A7).
- [ ] Tras una corrida de ingesta, **recargar el detalle del producto refleja el precio nuevo**
      sin rebuild ni espera de revalidación (ADR-0007, enmienda A2).
- [ ] El detalle no muestra ninguna funcionalidad de alerta de precio (retirada de v1).

### Historial de precios

- [ ] El gráfico del detalle muestra la serie de capturas válidas de al menos las últimas 4
      semanas cuando el producto lleva ese tiempo trackeado (con la semilla, desde el día uno).
- [ ] Si el producto lleva menos de 4 semanas, el gráfico muestra el rango disponible indicando
      desde cuándo hay datos, sin extrapolar.
- [ ] El historial distingue las series por tienda (o permite verlas por tienda), de modo que una
      caída en una tienda no se confunda con la tendencia de otra.

### Dashboard de tendencias

- [ ] "Mayores caídas de precio" se calcula **por `ListadoTienda`** (producto + tienda), no
      agregando por producto: ordena por mayor descenso porcentual entre la primera y la última
      captura válida del listado en los últimos 7 días, y cada fila nombra producto, tienda y
      porcentaje (hallazgo S2).
- [ ] "Más buscados" agrupa `BusquedaRegistrada` **por término normalizado** en los últimos 7
      días: "laptop", "Laptop" y "laptops" cuentan como una sola entrada. Cuando todos los
      registros del grupo apuntan al mismo `producto_id`, se renderiza la tarjeta de producto del
      wireframe 2c; si no, el término como texto (ADR-0003, enmienda A8).
- [ ] El índice de precios por categoría se calcula con la **fórmula de canasta fija** definida
      arriba (mismos productos en `t` y `t−7d`, media de variaciones relativas), muestra el número
      de productos de la canasta, y marca la categoría como "datos insuficientes" si la canasta
      tiene menos de 3 productos. Añadir un producto caro al catálogo **no** altera el índice
      (hallazgo A5).
- [ ] Si una sección del dashboard no tiene datos suficientes (ej. sin búsquedas registradas),
      muestra su estado vacío propio sin romper las demás secciones.

### Navegación por categorías

- [ ] La pantalla de categorías muestra las categorías del catálogo y cada una lleva a la grilla
      de sus productos.
- [ ] Una categoría sin productos muestra estado vacío claro (no una grilla rota).

### Ingesta de datos (semilla + scraper)

- [ ] El generador de semilla puede poblar desde cero un catálogo con las 4 tiendas, categorías,
      productos con listados en 2+ tiendas y 4+ semanas de historial con variaciones realistas,
      de forma repetible (un comando).
- [ ] Las capturas sembradas se fechan relativas a `now()` en cada corrida: el historial se
      extiende 4+ semanas hacia atrás desde el momento de la ejecución y la captura más reciente
      de cada listado cae dentro de las últimas 24 h. Ejecutar el seed y abrir el detalle de
      cualquier producto muestra "mejor oferta vigente" resaltada, sin badges de desactualizado
      (ADR-0001).
- [ ] El generador emite el historial en orden cronológico ascendente, de modo que la validación
      del ADR-0008 compare cada captura contra la anterior real de la serie.
- [ ] El generador siembra también `BusquedaRegistrada`: búsquedas distribuidas en los últimos 7
      días con frecuencias desiguales, de modo que "más buscados" tenga contenido en una
      instalación fresca (ADR-0003, enmienda A8).
- [ ] El scraper captura precios reales de al menos 1 tienda para listados ya curados y los
      persiste **por el mismo módulo de ingesta** que la semilla (misma validación, mismo
      esquema).
- [ ] Los jobs de ingesta corren **automáticamente, sin intervención manual, ~2 veces al día**,
      mediante el workflow programado de GitHub Actions (ADR-0010); el historial de ejecuciones
      del workflow es la evidencia. Una corrida fallida del scraper no afecta la disponibilidad de
      la web (solo envejece el dato, que se marca como desactualizado a las 24 h).
- [ ] El workflow puede lanzarse a mano (`workflow_dispatch`) como camino de respaldo antes de una
      demo.
- [ ] Toda captura pasa por la validación del ADR-0008 —estructural siempre, de desviación solo si
      hay captura previa—; las rechazadas quedan persistidas con flag de inválida para auditoría.

## Riesgos técnicos abiertos

- **Viabilidad real del scraping:** las 4 tiendas usan protecciones anti-bot y el PRD señala
  riesgo legal/ToS. El diseño lo acota (scraper best-effort en 1-2 tiendas, demo independiente de
  él — ADR-0001) pero no lo resuelve: si ninguna tienda resulta scrapeable, v1 queda solo con
  datos semilla. **Agravante nuevo (ADR-0010):** correr desde GitHub Actions significa salir por
  IP de datacenter conocida y fácil de bloquear, lo que aumenta la probabilidad de fallo frente a
  ejecutar en local. Antes de concluir que una tienda "no es scrapeable", conviene probarla
  también fuera del runner.
- **Matching de productos entre tiendas:** pospuesto por diseño (catálogo canónico curado,
  ADR-0003). Escalar a catálogo abierto requiere la investigación de matching que el PRD ya
  identificaba como riesgo central.
- **Picos de tráfico tipo CyberWow:** fuera del diseño de v1 por decisión de alcance (ADR-0008);
  la arquitectura no está dimensionada ni probada para tráfico masivo.
- **Umbrales de validación sin evidencia:** tanto el ±80% de desviación como el rango absoluto de
  plausibilidad (`S/ 1`–`S/ 100,000`, ADR-0008 enmienda A3) son valores iniciales elegidos a
  criterio. El primero puede rechazar ofertas legítimas agresivas; el segundo es una cota del
  catálogo actual, no una verdad del dominio. Revisar ambos con las capturas inválidas auditadas.
- **Ancla falsa en la primera captura:** la enmienda A3 del ADR-0008 reduce el riesgo con el rango
  absoluto, pero no lo elimina — una captura errónea que caiga dentro del rango sigue pudiendo
  fijarse como referencia de un listado. Eliminarlo exigía confirmación en dos lecturas,
  descartada por alcance.
- **Programación de los jobs no garantizada:** GitHub Actions encola los cron sin garantía de
  puntualidad y desactiva los workflows programados tras 60 días de inactividad del repositorio
  (ADR-0010). Aceptable en el horizonte del curso; primera decisión a revisar si el proyecto
  sigue vivo.
- **Sin caché en las rutas de precio:** la política del ADR-0007 (enmienda A2) hace que cada carga
  del detalle o de resultados golpee la base. Coherente con que los picos de tráfico estén fuera
  de alcance, pero es el punto que cede primero si el tráfico crece.
- **Registro de búsquedas dependiente de JavaScript:** al mover el registro al submit del cliente
  (ADR-0004, enmienda A1), las búsquedas hechas sin JS o por acceso directo a la URL no se
  cuentan. "Más buscados" subestima, y se prefirió eso a sobrestimar.
- **Foco de usuario del PRD sin resolver del todo:** el PRD deja abierto si prioriza al comprador
  puntual o al que monitorea en el tiempo; los wireframes priorizan al comprador puntual y este
  diseño lo sigue, pero un giro hacia monitoreo reabriría la decisión de alertas.
