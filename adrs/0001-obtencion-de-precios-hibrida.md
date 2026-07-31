# ADR 0001: Obtención de precios híbrida (datos semilla + scraper best-effort)

## Estado

Aceptado — enmendado el 2026-07-30 (anclaje temporal de la semilla) y el 2026-07-31 (procedencia
visible del dato). Ver las enmiendas al final.

## Contexto

El PRD identifica la obtención de precios como el mayor riesgo técnico del proyecto: "Se asume
que existe una forma viable (scraping, API pública o acuerdo) de obtener precios de las tiendas
online peruanas objetivo; esto no está confirmado", y señala explícitamente el "riesgo legal/de
términos de servicio al extraer precios de tiendas de terceros vía scraping". Además, dos
criterios de éxito dependen directamente de la ingesta de datos: la "mejor oferta vigente" debe
corresponder al precio más bajo capturado en las últimas 24 horas, y el historial de precios debe
mostrar al menos 4 semanas de datos.

Restricciones reales: proyecto académico (entregable de curso DMC), equipo de una persona, 4
tiendas objetivo (Falabella, Ripley, Plaza Vea, Oechsle) que emplean protecciones anti-bot y
cambian su HTML sin aviso. La demo del curso no puede depender de que un scraper externo funcione
el día de la presentación.

## Decisión

La plataforma se alimenta de dos fuentes que comparten un único pipeline de ingesta: (1) un
generador de datos semilla que produce productos, precios por tienda e historial de 4+ semanas
con variaciones realistas, garantizando que la plataforma siempre tenga datos completos; y (2) un
scraper real best-effort para 1-2 de las tiendas más accesibles, que demuestra la viabilidad de
la ingesta real inyectando sus capturas por el mismo pipeline que la semilla.

**Anclaje temporal de la semilla (enmienda 2026-07-30):** el generador no produce fechas fijas.
En cada ejecución calcula todas las marcas de tiempo **relativas a `now()`**: el historial se
extiende hacia atrás desde el instante de la corrida (4+ semanas) y la captura más reciente de
cada listado cae dentro de las últimas horas — dentro de la ventana de frescura de 24 h del
ADR-0008. El generador se ejecuta **como comando aparte** (`seed`), no en el cron de 12 h.

## Alternativas consideradas

- **Scraping propio de las 4 tiendas** — Viable y con datos 100% reales, pero las 4 tiendas usan
  protecciones anti-bot (Cloudflare/Akamai) y cambian su marcado sin aviso: el mantenimiento es
  alto para una persona y un scraper roto justo antes de la entrega dejaría la demo sin datos.
  Agrava el riesgo de términos de servicio ya señalado en el PRD.
- **Dataset simulado puro** — La opción más segura (sin riesgo legal ni dependencia externa),
  pero no valida el riesgo técnico central que el propio PRD marca como el mayor del proyecto:
  demostraría la plataforma sin probar que la ingesta real es posible.

## Consecuencias

- La demo del curso nunca depende de un servicio externo: los datos semilla garantizan catálogo,
  comparación e historial completos desde el día uno.
- El scraper real prueba el concepto de ingesta sobre el riesgo técnico central, acotado a 1-2
  tiendas para limitar mantenimiento y exposición a términos de servicio.
- **Trade-off:** se mantienen dos productores de datos (generador y scraper). Ambos deben escribir
  por el mismo pipeline y contrato de ingesta; si divergen, se convierten en dos sistemas
  distintos y el scraper deja de demostrar nada sobre la plataforma real.
- Los precios sembrados no son reales: la plataforma debe dejar claro (disclaimer ya previsto en
  el footer del wireframe) que los precios son referenciales.

## Enmienda 2026-07-30: anclaje temporal de la semilla

**Qué cambió:** se añade a la decisión que el generador siembra timestamps relativos a `now()` en
cada ejecución, y se fija que corre como comando aparte y no dentro del cron de 12 h.

**Por qué:** la revisión adversarial (`REVISION-ADVERSARIAL.md`, hallazgo C1) detectó que esta
decisión y el ADR-0008 se anulaban mutuamente. La promesa de este ADR es que "la demo del curso
nunca depende de un servicio externo: los datos semilla garantizan catálogo, comparación e
historial completos desde el día uno". El ADR-0008 excluye del cálculo de "mejor oferta vigente"
toda captura de más de 24 h y la marca como desactualizada. Con fechas fijas, una semilla
generada el lunes y presentada el jueves deja **todas** las capturas fuera de la ventana: ningún
listado califica para mejor oferta, el resaltado "Mejor precio · Tienda" del wireframe 2a no
aparece en ninguna pantalla, y el dashboard de 7 días sí sigue funcionando — un fallo parcial,
más difícil de diagnosticar que una caída total. Es decir: la promesa central de este ADR se
rompía sola con el paso de los días.

**Alternativas de la enmienda:**

- **Correr el generador dentro del cron de 12 h** — Mantendría los datos frescos sin intervención
  manual, pero mezcla dos productores en la misma cadencia: el generador reescribiría o
  extendería la historia sembrada en paralelo a las capturas reales del scraper, y dejaría de
  quedar claro qué parte del historial es real. Además la semilla es idempotente por diseño
  (poblar desde cero), no incremental como el scraper.
- **Rebajar o quitar la ventana de 24 h** — Arreglaría la demo tocando el ADR-0008, pero contradice
  un criterio de éxito explícito del PRD ("la mejor oferta corresponde al precio más bajo
  capturado en las últimas 24 horas"). Descartada: el problema está en la semilla, no en la regla.

**Consecuencias de la enmienda:**

- La demo muestra datos vigentes se ejecute el seed el mismo día de la presentación o tres
  semanas antes, siempre que el seed se haya corrido después del último `reset` de la base.
- **Trade-off:** hay un paso operativo que recordar. Como el generador no corre en el cron, una
  base sembrada hace días vuelve a envejecer: la mitigación real es que `seed` sea barato y
  repetible (un comando, ya exigido en los criterios de aceptación) y que se ejecute antes de
  cada demo. Se acepta a cambio de no contaminar la serie real con datos generados.
- El generador debe emitir el historial en orden cronológico ascendente para que la validación
  del ADR-0008 compare cada captura contra la anterior real y no contra un punto arbitrario de la
  serie (requisito recogido después en la enmienda del ADR-0008, hallazgo A3).

## Enmienda 2026-07-31: procedencia visible del dato

**Qué cambió:** el campo `fuente` de `CapturaPrecio` (`semilla` | `scraper`, ADR-0003) deja de ser
un dato solo interno y **se refleja en la interfaz**:

- Toda fila de precio cuya última captura válida tenga `fuente = semilla` muestra un indicador
  visible junto al precio — etiqueta **"dato simulado"**, con el mismo tratamiento de badge que el
  "precio desactualizado" del ADR-0008 y tooltip que explica que no es un precio capturado de la
  tienda.
- El disclaimer del footer ("Precios referenciales, verifica en tienda", DESIGN.md) se mantiene,
  pero deja de ser la única señal.
- Cuando **todas** las capturas mostradas en una pantalla son de semilla, la pantalla muestra
  además un aviso a nivel de página, no solo por fila.
- El campo `fuente` se expone en las respuestas de la API (ADR-0004) para que el frontend pueda
  renderizarlo.

**Por qué:** la revisión adversarial (`REVISION-ADVERSARIAL.md`, hallazgo A7) señaló un riesgo que
ningún ADR había nombrado y que es **distinto** del riesgo de scraping/ToS que este ADR sí mitiga:
el modelo del ADR-0003 guarda `Tienda` con nombre y logo, y las cuatro son empresas reales, así
que una fila que dice "Falabella — S/ 2,499" con su logo **atribuye a un negocio identificado un
precio que la plataforma inventó**. Extraer un precio real sin permiso y publicar uno falso a
nombre de otro no son el mismo problema, y un disclaimer genérico en el footer no cubre la
atribución nominal.

Hay además una razón de honestidad de la demo, no solo de riesgo: sin esta distinción, el
evaluador del curso **no puede saber qué precio vino del scraper real y cuál lo generó el seed**,
que es justamente lo que este ADR pretende demostrar. Marcar la procedencia hace visible el
resultado del scraper en lugar de diluirlo entre datos sintéticos.

**Alternativas de la enmienda:**

- **Usar nombres genéricos ("Tienda A", "Tienda B") en los datos sembrados** — Elimina el problema
  de raíz: sin nombre real no hay atribución. Descartada porque destruye el realismo que hace útil
  la demo (el wireframe resalta "Mejor precio · Tienda" con marcas reconocibles) y porque
  volvería incomparables las filas de semilla con las del scraper real, que sí llevan tienda real:
  la pantalla de comparación mezclaría "Tienda A" con "Falabella".
- **Dejar solo el disclaimer del footer** — Cero trabajo, y es lo que el diseño ya tenía.
  Descartada porque un disclaimer global no distingue qué precio concreto es inventado; con la
  estrategia híbrida de este ADR conviven en la misma tabla precios reales y simulados, y esa es
  exactamente la distinción que el usuario necesita.

**Consecuencias de la enmienda:**

- La plataforma nunca presenta un precio inventado como si fuera capturado de la tienda, y el
  aporte del scraper real queda visible por contraste.
- **Trade-off:** la pantalla estrella pierde limpieza. Mientras el scraper cubra 1-2 tiendas
  (alcance de este ADR), la mayoría de las filas llevarán el badge de "dato simulado", lo que
  resta impacto visual a la comparación. Se acepta: una demo honesta que se ve algo más cargada es
  preferible a una demo limpia que atribuye precios falsos a empresas reales.
- **Trade-off:** aparece un segundo badge que puede coincidir con el de "precio desactualizado"
  (ADR-0008) en la misma fila. El diseño visual tiene que resolver esa combinación; el DESIGN.md
  ya tenía el estado de desactualizado como pendiente y ahora hereda también este.
