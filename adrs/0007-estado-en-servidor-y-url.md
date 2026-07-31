# ADR 0007: Estado en servidor con la URL como estado de interfaz

## Estado

Aceptado — enmendado el 2026-07-30 (ver "Enmienda 2026-07-30: las páginas consumen la API del
ADR-0004").

## Contexto

La fuente de verdad de los datos ya está resuelta: PostgreSQL alimentado por la ingesta (ADRs
0002 y 0006), y el PRD excluye cuentas de usuario y personalización, así que no existe estado de
sesión. Queda decidir dónde vive el estado de la interfaz — término buscado, filtros de
tienda/precio/orden de la pantalla de resultados (2a), producto abierto — y cómo llegan los datos
del servidor a las pantallas construidas con Next.js (ADR-0005).

## Decisión

Las pantallas se renderizan en el servidor con los datos ya cargados (Server Components / SSR que
obtienen los datos **llamando a los endpoints REST del ADR-0004 mediante `fetch` server-side**),
y todo estado de interfaz relevante vive en la URL
como query params (ej. `/buscar?q=laptop&tienda=ripley&orden=precio`). En memoria del cliente
solo queda estado efímero: el texto en el input del buscador antes de enviar y las interacciones
del gráfico de historial.

## Alternativas consideradas

- **Fetching en cliente con SWR/React Query** — Interacciones sin recarga y caché entre
  pantallas, pero duplica el estado (URL vs caché de cliente) obligando a sincronizarlos a mano,
  y por defecto las URLs dejan de reflejar lo que el usuario ve — grave en un comparador, donde
  compartir el enlace de una comparación es el caso de uso natural.
- **Store global en cliente (Zustand/Redux)** — Estado centralizado y predecible, pero sin
  sesión de usuario ni estado compartido entre pantallas que lo justifique: añade maquinaria y
  boilerplate que las otras opciones evitan por completo.

## Consecuencias

- Toda vista es compartible y recargable por URL (búsquedas, filtros, producto), y el botón
  atrás del navegador funciona sin código adicional.
- Mínima superficie de estado que mantener: no hay caché de cliente ni store que invalidar
  cuando la ingesta actualiza precios — cada carga lee datos frescos del servidor. *(Matizado en
  la enmienda del 2026-07-31: esto no ocurre por defecto en Next.js; exige la política de caché
  explícita que allí se decide.)*
- **Trade-off:** cada cambio de filtro u orden implica una navegación y re-render del servidor;
  interacciones muy finas (reordenar resultados sin parpadeo perceptible) exigen cuidado
  (transiciones de React/Next) y siempre habrá más latencia que con una caché local.

## Enmienda 2026-07-30: las páginas consumen la API del ADR-0004

**Qué cambió:** la decisión decía que los Server Components consultaban "la capa de datos del
ADR-0004". Ahora dice explícitamente que la obtienen **llamando a los endpoints REST del ADR-0004
por `fetch` server-side**. La API es el único camino de datos de la aplicación; no existe una
segunda ruta que vaya de las páginas a la base de datos.

**Por qué:** la revisión adversarial (`REVISION-ADVERSARIAL.md`, hallazgo C2) señaló que este ADR
y el ADR-0004 se contradecían. El ADR-0004 descartó explícitamente la alternativa "renderizado en
servidor sin API formal", y la redacción anterior de este ADR describía justamente eso: si un
Server Component consulta la base directamente, los endpoints `/api/*` se quedan sin consumidor y
pasan a ser código muerto que duplica la lógica de las páginas. Había que elegir una de las dos
lecturas, y se eligió esta: **la API es el camino de datos real**, no una superficie paralela
mantenida solo para poder enseñarla en la evaluación. Así la consecuencia que el ADR-0004 se
atribuye —un contrato explícito y verificable con curl/Postman— describe el sistema que
efectivamente corre, y no una fachada.

**Alternativa de la enmienda:** resolver la contradicción por el otro lado, es decir, aceptar que
las páginas consultan la base directamente y degradar la API a superficie de evaluación. Es menos
plomería y una llamada de red menos por render, pero deja dos implementaciones de la misma
consulta que pueden divergir en silencio: la que el evaluador prueba con curl no sería la que el
usuario ve en pantalla. Descartada por eso.

**Consecuencias de la enmienda:**

- Una sola implementación de cada consulta. Lo que devuelve `GET /api/productos/{id}` es
  exactamente lo que renderiza la página de detalle.
- **Trade-off:** un salto HTTP interno adicional por render. En una webapp de un solo deploy
  (ADR-0002) es tráfico local y el costo es pequeño, pero no es cero, y obliga a que el `fetch`
  server-side resuelva su propia URL base (absoluta) según el entorno.
- **Trade-off:** al pasar por `fetch`, las respuestas entran en el caché de datos de Next.js. La
  afirmación de este ADR de que "cada carga lee datos frescos del servidor" **no se sostiene por
  defecto** y exige configuración explícita (`cache: 'no-store'` o `revalidate` corto) en los
  endpoints de precio. Este punto ya estaba señalado como hallazgo A2 y esta enmienda lo vuelve
  más agudo, no menos. **Resuelto en la enmienda del 2026-07-31**, abajo.

## Enmienda 2026-07-31: política de caché explícita por tipo de ruta

**Qué cambió:** este ADR afirmaba en sus consecuencias que "cada carga lee datos frescos del
servidor" **sin decidir nada que lo garantizara**. Ahora la política de caché es una decisión
explícita, por tipo de ruta:

| Rutas | Política | Configuración |
|---|---|---|
| Detalle de producto, resultados de búsqueda, y sus endpoints `/api/productos/*`, `/api/busqueda` | **Siempre dinámicas** — nunca sirven precio cacheado | `export const dynamic = 'force-dynamic'` en la ruta + `fetch(..., { cache: 'no-store' })` |
| Dashboard de tendencias (`/tendencias`, `/api/tendencias`) | Revalidación corta | `export const revalidate = 300` (5 min) |
| Categorías y catálogo (`/categorias`, `/api/categorias/*`) | Revalidación larga | `export const revalidate = 3600` (1 h) |

La regla que las une: **toda ruta que muestre un precio vigente o un badge de frescura es
dinámica**. Las demás pueden cachearse porque su contenido (nombres de categoría, composición del
catálogo) cambia por curación manual, no por ingesta.

**Por qué:** la revisión adversarial (`REVISION-ADVERSARIAL.md`, hallazgo A2) señaló que el App
Router de Next.js hace lo contrario de lo que este ADR daba por sentado: renderiza estáticamente
lo que puede y cachea `fetch` por defecto. Una página de producto sin configuración explícita
puede servir el precio del momento del build. Es exactamente el fallo que el ADR-0008 existe para
prevenir —mostrar un dato obsoleto como si fuera vigente— reintroducido por el framework elegido
en el ADR-0005, y por debajo de la capa donde el ADR-0008 puede verlo: la validación y el badge de
"desactualizado" son correctos, pero llegan al usuario dentro de un HTML viejo. La enmienda del
C2 (fetch server-side) lo agudizó, porque metió las respuestas de precio en el caché de datos.

**Alternativas de la enmienda:**

- **`force-dynamic` en toda la aplicación** — Una sola regla, imposible de olvidar, y a la escala
  de este proyecto el costo sería tolerable. Descartada porque tira también el caché de las
  pantallas que sí se benefician (categorías, catálogo) y borra la distinción útil entre datos que
  cambian por ingesta y datos que cambian por curación.
- **`revalidate` corto (ej. 60 s) también en el detalle de producto** — Menos carga en la base y
  una ventana de desactualización pequeña frente a la de 24 h del ADR-0008. Descartada porque el
  producto muestra explícitamente la fecha/hora de la última captura: servir HTML cacheado
  significaría mostrar un timestamp que no corresponde al estado real, que es peor que un precio
  algo viejo — el ADR-0008 vende frescura visible, y una mentira sobre la frescura es más grave
  que la falta de frescura.

**Consecuencias de la enmienda:**

- La afirmación original de este ADR ("cada carga lee datos frescos") pasa a ser cierta donde
  importa, pero **por configuración, no por defecto**: es una decisión que hay que sostener en cada
  ruta nueva que muestre precio.
- **Trade-off:** las rutas de precio pierden todo caché y golpean la base en cada carga. Aceptable
  al tráfico de v1, y coherente con que los picos tipo CyberWow ya están fuera de alcance
  (ADR-0008); si el tráfico creciera, esta es la primera decisión a revisar.
- **Trade-off:** tres políticas distintas en vez de una es más superficie que recordar. Se mitiga
  con la regla simple de arriba (¿muestra precio o frescura? → dinámica) en vez de una lista de
  rutas que haya que mantener.
- Criterio de aceptación añadido en `TECH-DESIGN.md`: tras una corrida de ingesta, recargar el
  detalle de producto refleja el precio nuevo sin rebuild ni espera.
