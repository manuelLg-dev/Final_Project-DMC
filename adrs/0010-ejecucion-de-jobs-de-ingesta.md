# ADR 0010: Ejecución de los jobs de ingesta (GitHub Actions programado)

## Estado

Aceptado (2026-07-31)

## Contexto

El ADR-0002 apoya buena parte de su valor en que "la ingesta corre y falla de forma aislada": un
scraper caído nunca tumba la web. El ADR-0005 elige Playwright para el scraper. El ADR-0006
propone PostgreSQL en Docker local o en un gestionado con capa gratuita (Supabase/Neon) "si el
proyecto se despliega". `TECH-DESIGN.md` fija que los jobs corren cada 12 h por tienda.

La revisión adversarial (`REVISION-ADVERSARIAL.md`, hallazgo A4) detectó que **falta la pieza que
une esas decisiones: qué proceso ejecuta el cron de 12 h**. Sin ella, el criterio de aceptación
"los jobs de ingesta corren cada 12 h" es literalmente inverificable, y la afirmación de
aislamiento del ADR-0002 no está respaldada por ninguna decisión de despliegue.

La restricción que decide el caso: **Playwright necesita un navegador y varios minutos**. El
destino natural de una app Next.js (Vercel) no sirve para esto — sus funciones tienen límites de
ejecución cortos y no incluyen navegador — así que el "un solo deploy" del ADR-0002 se refiere a
la webapp, no al pipeline. Son dos entornos de ejecución, aunque sea un solo repositorio.

## Decisión

Los jobs de ingesta corren en **GitHub Actions con `schedule`**, en un workflow propio del mismo
repositorio:

- Un workflow `ingesta.yml` con `on: schedule` (dos corridas diarias, ~00:00 y ~12:00 UTC) y
  `on: workflow_dispatch` para poder lanzarlo a mano.
- El job instala dependencias, ejecuta el scraper contra las tiendas configuradas y escribe en la
  base gestionada (Neon/Supabase) mediante una `DATABASE_URL` guardada en los secrets del
  repositorio.
- El **generador de semilla no corre aquí**: es un comando local previo a la demo (ADR-0001,
  enmienda 2026-07-30).

**Sobre el "cada 12 h": es real, pero con una salvedad honesta.** GitHub Actions no garantiza
puntualidad —los cron programados se encolan y pueden retrasarse en momentos de carga— y
**deshabilita los workflows programados tras 60 días sin actividad en el repositorio**. Para el
horizonte de un entregable de curso ambas cosas son tolerables, pero el criterio correcto no es
"corre exactamente cada 12 h" sino **"corre automáticamente ~2 veces al día, sin intervención
manual, y una corrida fallida no afecta a la web"**. Ese es el criterio que se lleva a
`TECH-DESIGN.md`.

La ejecución manual (`workflow_dispatch`, o el script en local) queda como camino de respaldo
explícito para la demo, no como el mecanismo principal.

## Alternativas consideradas

- **Ejecución manual antes de cada demo** — Es la respuesta honesta mínima y hay que reconocer que
  bastaría: la enmienda del ADR-0001 ya garantiza que el seed produce datos frescos cuando se
  ejecuta, así que la demo no necesita ningún cron para verse bien. Descartada porque el ADR-0001
  justifica el scraper como prueba de que "la ingesta real es viable", y un scraper que solo corre
  cuando alguien lo lanza a mano no demuestra ingesta continua: demuestra un script. La diferencia
  es justamente lo que el proyecto quiere enseñar. Se conserva como respaldo, no como plan.
- **Contenedor propio en Fly.io / Railway con cron interno** — El entorno más parecido a
  producción real, con proceso de larga duración y sin límites de tiempo. Descartada por costo de
  operación desproporcionado: añade un segundo servicio que desplegar, monitorear y mantener vivo
  (más el Dockerfile con las dependencias de Playwright), para dos ejecuciones diarias de pocos
  minutos. Con capa gratuita además suele haber suspensión por inactividad, que reintroduce el
  problema por otro lado.
- **Vercel Cron Functions** (mismo deploy que la webapp) — Lo más simple si funcionara, y mantiene
  todo en un solo sitio. Descartada por incompatibilidad técnica real, no por preferencia: no hay
  navegador disponible y el límite de ejecución no alcanza para un scraper con Playwright.
  Serviría solo si el scraping fuera HTTP + Cheerio, que no es lo que el ADR-0005 dejó decidido.

## Consecuencias

- El criterio de "corre automáticamente" pasa a ser **verificable**: el historial de ejecuciones
  del workflow en GitHub es la evidencia, y sirve como prueba en la evaluación del curso.
- Cero infraestructura nueva que operar: se usa el mismo repositorio donde ya vive el código.
- El aislamiento que promete el ADR-0002 se vuelve real y no solo conceptual — el scraper corre en
  una máquina distinta de la webapp, así que un job colgado o un Playwright que muere no puede
  afectar a la web ni consumir sus recursos.
- **Trade-off:** GitHub Actions no es un scheduler puntual. Las corridas pueden retrasarse y los
  workflows programados se desactivan tras 60 días de inactividad del repositorio. Para v1 es
  aceptable; si el proyecto siguiera vivo, esta es la primera decisión a revisar.
- **Trade-off:** la base de datos queda expuesta a internet para que el runner de Actions pueda
  escribir. Exige una `DATABASE_URL` en secrets y credenciales acotadas a la ingesta; es una
  superficie que la alternativa "todo local" no tenía.
- **Trade-off:** las IP de los runners de GitHub son conocidas y fáciles de bloquear para un
  sistema anti-bot. Esto **aumenta la probabilidad de que el scraper falle** frente a ejecutarlo
  desde una IP residencial. Se acepta porque el ADR-0001 ya diseñó el sistema para que la demo no
  dependa del scraper, y porque un scraper que falla en Actions sigue siendo observable (el log
  queda) — pero conviene saberlo antes de concluir que "una tienda no es scrapeable".