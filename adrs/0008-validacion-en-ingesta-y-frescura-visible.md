# ADR 0008: Validación de precios en la ingesta y frescura visible en la web

## Estado

Aceptado — enmendado el 2026-07-31 (ver "Enmienda 2026-07-31: primera captura de un listado").

## Contexto

Los casos borde del PRD giran alrededor de la confiabilidad del dato mostrado: "indicar la
fecha/hora de la última actualización de precio en vez de mostrar un dato silenciosamente
obsoleto", "precio con error evidente de captura (ej. precio en 0 o absurdamente bajo/alto...)
no debe mostrarse como 'mejor oferta vigente' sin validación mínima", y el criterio de éxito que
define la mejor oferta como el precio más bajo capturado en las últimas 24 horas. La arquitectura
ya provee los puntos de anclaje: un módulo de ingesta común por el que pasan semilla y scraper
(ADR-0001/0002) y un flag de validez en `CapturaPrecio` (ADR-0003).

## Decisión

La validación vive en el módulo de ingesta común, única puerta de entrada de datos:

- Toda captura se valida antes de persistirse: precio mayor que 0 y desviación acotada frente a
  la última captura válida del mismo listado (umbral inicial: rechazo de saltos mayores a ±80%).
  La regla de desviación **solo aplica cuando existe una captura válida previa** del mismo
  listado; ver la enmienda del 2026-07-31 para el caso de la primera captura.
- Las capturas que fallan la validación se guardan igualmente con el flag de invalidez activo:
  quedan para auditoría pero nunca compiten por la mejor oferta ni aparecen en el historial.
- La web muestra siempre la fecha/hora de la última captura válida por tienda; si supera las 24
  horas, el precio se marca con un badge de "precio desactualizado" (estado ya identificado como
  pendiente en DESIGN.md) y queda excluido del cálculo de "mejor oferta vigente".

Los picos de tráfico tipo CyberWow quedan explícitamente fuera del diseño de v1 por decisión de
alcance (proyecto de curso); se documentan como riesgo conocido en TECH-DESIGN.md.

## Alternativas consideradas

- **Guardar todo crudo y validar en lectura** — No pierde ningún dato y permite cambiar reglas
  sin re-ingestar, pero obliga a repetir (o compartir con disciplina) la lógica de filtrado en
  cada endpoint; un descuido en una sola consulta mostraría el precio en 0 como mejor oferta —
  exactamente lo que el PRD prohíbe.
- **Solo frescura, sin validación de outliers** — La opción más simple, pero incumple
  directamente el caso borde del PRD sobre precios con error evidente de captura. Descartada por
  contradecir un requisito explícito.

## Consecuencias

- Hay una sola puerta donde razonar sobre calidad de datos, independiente de la fuente (semilla o
  scraper), y las consultas de la web quedan simples: filtrar por flag de validez y ventana de
  24 horas.
- Las capturas rechazadas quedan auditables (se guardan marcadas), lo que permite ajustar el
  umbral con evidencia real.
- **Trade-off:** un umbral fijo de desviación puede rechazar ofertas reales agresivas — una
  rebaja legítima de CyberWow podría marcarse como sospechosa y no mostrarse hasta ajustar el
  umbral o revalidarla. Es el costo asumido de nunca mostrar un precio absurdo como mejor oferta.
- La limitación de no estar dimensionado para picos masivos de tráfico queda registrada como
  riesgo abierto, no resuelta.

## Enmienda 2026-07-31: primera captura de un listado

**Qué cambió:** la regla de validación se desdobla en dos reglas explícitas según haya o no una
captura válida previa del mismo listado:

1. **Validación estructural (siempre aplica):** el precio debe ser un número finito **mayor que 0**
   y estar dentro de un rango absoluto de plausibilidad del catálogo (cota inicial: `S/ 1` a
   `S/ 100,000`). Se aplica a toda captura, tenga o no historial.
2. **Validación de desviación (solo con referencia):** el rechazo por salto mayor a ±80% se evalúa
   **únicamente si existe una captura válida previa** del mismo listado. La primera captura de un
   listado no tiene contra qué compararse y por tanto **no se somete a esta regla**: si pasa la
   validación estructural, se acepta y se convierte en la referencia de la siguiente.

Además, la captura de referencia es siempre **la captura válida inmediatamente anterior en orden
cronológico**, no la última insertada. El módulo de ingesta procesa cada listado en orden temporal
ascendente.

**Por qué:** la revisión adversarial (`REVISION-ADVERSARIAL.md`, hallazgo A3) señaló que este ADR
definía la validación como "desviación frente a la última captura válida del mismo listado" sin
decir qué ocurre cuando no hay ninguna. El hueco es real y es peor de lo que parece: quedaba
implícito que la primera captura se acepta sin más, con lo que un scraper que arranca leyendo mal
el HTML **establece la línea base con un valor falso** — y entonces la segunda captura, la
correcta, se rechaza por desviarse de él. La regla resultaba más frágil en el arranque que en
régimen, justo cuando el scraper es menos confiable, y podía dejar un listado bloqueado en un
precio erróneo indefinidamente. La cota absoluta de plausibilidad es lo que rompe ese ciclo sin
necesitar historial.

El segundo punto (orden cronológico) ya se había hecho requisito en la enmienda del ADR-0001 para
que la semilla no se autoinvalide; aquí queda escrito como parte de la regla de validación, que es
donde pertenece.

**Alternativas de la enmienda:**

- **Aceptar cualquier precio > 0 en la primera captura, sin cota absoluta** — Lo más simple y lo
  que el ADR ya insinuaba. Descartada porque deja intacto el problema del ancla falsa: es
  precisamente en la primera captura donde no hay ninguna defensa, y basta un selector CSS mal
  apuntado (que capture un precio de cuota mensual, un código de producto o un número de reseñas)
  para envenenar la serie entera de ese listado.
- **Marcar la primera captura como "provisional" y confirmarla con la segunda** — Más robusto: dos
  lecturas coherentes antes de dar un precio por bueno. Descartada por desproporción con el
  alcance de curso: retrasa 12 h la aparición de todo listado nuevo, añade un tercer estado a
  `CapturaPrecio` (además del flag de validez) y complica todas las consultas de la web para
  cubrir un caso que la cota absoluta ya acota razonablemente.

**Consecuencias de la enmienda:**

- Existe una defensa mínima en el arranque de cada listado, que es donde antes no había ninguna, y
  el modo de fallo "el primer error queda como referencia permanente" desaparece de los casos
  groseros (precio en 0, un `12` que era el número de cuotas, un `20250731` que era un id).
- La regla queda enunciada de forma que se puede implementar y probar directamente: dos funciones,
  una sin dependencia del historial y otra que la requiere.
- **Trade-off:** el rango absoluto (`S/ 1`–`S/ 100,000`) es tan arbitrario como el ±80% y comparte
  su destino — habrá que revisarlo con las capturas inválidas auditadas. Un producto legítimo
  fuera de ese rango se rechazaría; con el catálogo curado de v1 (electrodomésticos y tecnología)
  no debería ocurrir, pero es una cota del catálogo actual, no una verdad del dominio.
- **Trade-off:** una captura errónea que caiga **dentro** del rango absoluto sigue pudiendo
  anclarse como primera referencia. La enmienda reduce el problema, no lo elimina; eliminarlo
  exigía la alternativa de confirmación en dos lecturas, descartada arriba.
