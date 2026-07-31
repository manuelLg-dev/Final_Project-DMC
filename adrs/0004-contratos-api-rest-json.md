# ADR 0004: API REST JSON interna como contrato entre frontend y datos

## Estado

Aceptado — enmendado el 2026-07-31 (ver "Enmienda 2026-07-31: el registro de búsqueda sale del
`GET`").

## Contexto

Con la arquitectura de dos componentes (ADR-0002), quedan dos interfaces por definir. El contrato
de ingesta ya quedó resuelto en los ADR-0001/0002: semilla y scraper escriben a la base de datos
a través de un módulo de ingesta común. La interfaz abierta es cómo el frontend de la aplicación
web obtiene los datos que sus pantallas necesitan: búsqueda (1a), comparación por tienda (2a),
detalle con historial (2b), tendencias (2c) y categorías (2d). Restricción: un solo cliente (web
desktop) en v1, pero el DESIGN.md deja "flujo mobile" como pendiente futuro.

## Decisión

La aplicación web expone una API REST JSON interna, poseída y documentada por el backend, que el
frontend consume. Endpoints principales:

- `GET /api/busqueda?q={termino}` — resultados de búsqueda. **Sin efectos secundarios** (el
  registro se separó a un endpoint propio; ver la enmienda del 2026-07-31).
- `POST /api/busquedas` — registra una búsqueda enviada por el usuario (ADR-0003).
- `GET /api/productos/{id}` — detalle del producto con precios vigentes por tienda y mejor oferta.
- `GET /api/productos/{id}/historial` — serie de precios para el gráfico.
- `GET /api/tendencias` — datos del dashboard (mayores caídas, más buscados, índice por categoría).
- `GET /api/categorias` y `GET /api/categorias/{id}/productos` — catálogo.

## Alternativas consideradas

- **Renderizado en servidor sin API formal** — Menos plomería (una función por pantalla), pero el
  contrato queda implícito en funciones internas, y las partes interactivas (gráfico de historial,
  filtros de resultados) terminarían necesitando endpoints ad-hoc de todos modos, mezclando dos
  estilos.
- **GraphQL** — Flexible para pantallas que agregan datos (el dashboard combina tres fuentes),
  pero sobredimensionado para un único cliente y cinco pantallas: añade esquema, servidor y
  tooling sin un segundo consumidor que lo amortice.

## Consecuencias

- El contrato es explícito y verificable con herramientas estándar (curl/Postman), lo que
  facilita demostrar y probar el backend por separado en la evaluación del curso.
- Si el proyecto escala a otro cliente (móvil, pendiente en DESIGN.md), la API ya existe y solo
  habría que extraerla del deploy (trade-off ya asumido en ADR-0002).
- **Trade-off:** cada pantalla requiere endpoint + llamada + estados de carga/error en el cliente;
  es más código que renderizar directo en servidor, y el contrato hay que mantenerlo sincronizado
  con el frontend a mano (sin tipado compartido garantizado, según el stack que se elija).

## Nota 2026-07-30: quién consume esta API

La revisión adversarial (`REVISION-ADVERSARIAL.md`, hallazgo C2) detectó que el ADR-0007, al
decidir renderizado en servidor, describía la alternativa que este ADR había descartado, dejando
sin definir si estos endpoints tenían consumidor real.

Resuelto en el ADR-0007 (enmienda 2026-07-30): **las páginas Server Component consumen estos
endpoints por `fetch` server-side**. Esta API es el único camino de datos de la aplicación, no una
superficie paralela para la evaluación.

Esto matiza el trade-off de arriba: no hay "estados de carga/error en el cliente" por pantalla,
porque la llamada ocurre en el servidor antes de renderizar; el costo real es un salto HTTP
interno por render y la necesidad de controlar el caché de `fetch` de Next.js. Ver las
consecuencias de la enmienda del ADR-0007.

Consecuencia pendiente: los criterios de aceptación de `TECH-DESIGN.md` están escritos solo en
términos de pantallas y URLs, y no verifican ningún endpoint. Conviene añadir criterios a nivel de
API ahora que es el camino de datos real.

## Enmienda 2026-07-31: el registro de búsqueda sale del `GET`

**Qué cambió:** `GET /api/busqueda` deja de tener efectos secundarios. El registro pasa a un
endpoint propio, `POST /api/busquedas`, invocado **una sola vez desde el cliente al enviar el
formulario** (evento de submit del buscador), no durante el render de la página de resultados. Es
una llamada `fire-and-forget`: su fallo no afecta a los resultados que el usuario ve.

**Por qué:** la revisión adversarial (`REVISION-ADVERSARIAL.md`, hallazgo A1) señaló que un `GET`
con escritura, combinado con las decisiones del ADR-0007 (URL compartible, recargable, botón atrás
funcional), infla el contador por construcción: cada F5, cada vuelta atrás, cada apertura de un
enlace compartido, cada crawler y cada **prefetch de `<Link>` de Next.js** —activo por defecto—
suma un registro. El criterio de aceptación decía "cada búsqueda **enviada** queda registrada",
pero bajo renderizado en servidor no existe un evento "enviada" distinguible de "página
renderizada": el único lugar donde esa distinción existe es el cliente, en el submit. Como el
dashboard "más buscados" es una de las cinco pantallas del producto, su métrica sería ruido de
navegación en lugar de intención de usuario.

Efecto colateral favorable: `GET /api/busqueda` queda idempotente, como corresponde a un `GET`, y
por tanto es cacheable y seguro de prefetchear (relevante para la política de caché del ADR-0007).

**Alternativas de la enmienda:**

- **Dejar la escritura en el `GET` y deduplicar** (por sesión, IP+término, o ventana de tiempo) —
  Evita depender de JavaScript en el cliente. Descartada porque para deduplicar bien hace falta
  identificar al visitante, y el PRD excluye cuentas de usuario; cualquier heurística por IP o
  cookie añade tratamiento de datos que este proyecto decidió no tener (`BusquedaRegistrada` es
  anónima por diseño, ADR-0003). Además no arregla el prefetch, que ocurre sin que nadie busque
  nada.
- **Server Action de Next.js en vez de endpoint REST** — Más directo y sin plomería de API.
  Descartada por coherencia con la enmienda del C2: se acaba de decidir que la API REST es el
  único camino de datos; abrir una segunda vía de escritura contradiría esa decisión el mismo mes.

**Consecuencias de la enmienda:**

- "Más buscados" mide intención real de búsqueda, no tráfico de navegación.
- **Trade-off:** el registro depende de JavaScript en el cliente. Una búsqueda hecha con JS
  deshabilitado (o por un cliente que llega directo a la URL con `?q=`) no se registra. Se acepta:
  perder registros es preferible a inflarlos, porque un "más buscados" inflado por prefetch es
  activamente engañoso mientras que uno incompleto solo es menos rico.
- **Trade-off:** una llamada de red más en el flujo de búsqueda. Al ser `fire-and-forget` no
  bloquea la navegación a la página de resultados.
