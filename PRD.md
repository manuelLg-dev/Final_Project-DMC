---
title: "AhorraPE (nombre por definir)"
---

# PRD: AhorraPE (nombre por definir)

## Problema

Los consumidores peruanos que quieren comprar un producto online no tienen forma rápida de saber
si el precio que ven es realmente el mejor disponible. Para comparar, tienen que abrir manualmente
varias tiendas online (o revisar promociones puntuales como el CyberWow) y comparar precios a mano,
lo cual consume tiempo y hace que muchas compras se cierren a un precio que no es el más
conveniente, o que se pierdan ofertas vigentes por no revisar a tiempo.

## Usuario objetivo

Compradores online en Perú que buscan un producto específico (electrodomésticos, tecnología, etc.)
y quieren confirmar, antes de comprar, cuál tienda ofrece el mejor precio en ese momento —
tanto compradores puntuales (una compra concreta) como usuarios que quieren monitorear el precio
de un producto en el tiempo para comprar en el momento óptimo.
<!-- REVISAR: validar si el foco inicial es "comprador puntual que busca hoy" o "usuario que monitorea precios en el tiempo" — ambos aparecen en la idea original y pueden implicar prioridades de producto distintas. -->

## Objetivo / resultado esperado

Un usuario puede llegar a la plataforma, buscar un producto desde un buscador central, y en una
sola pantalla ver el precio de ese producto en las distintas tiendas online de Perú que lo venden,
identificar cuál es la mejor oferta vigente, y revisar cómo ha evolucionado el precio en el tiempo —
sin necesidad de visitar cada tienda por separado.

## Alcance (qué sí incluye esta versión)

- Buscador central como pantalla principal para encontrar un producto por nombre/palabra clave.
- Comparación de precios de un mismo producto entre distintas tiendas online de Perú.
- Organización de productos por categorías.
- Vista de detalle de producto con historial de precios (tendencia en el tiempo).
- Dashboard de tendencias de precios (visión general, no solo por producto individual).
- Identificación visible de la "mejor oferta vigente" para un producto dado.
<!-- REVISAR: definir de cuántas y cuáles tiendas se obtendrán datos en esta primera versión (lista concreta de tiendas soportadas), ya que esto determina el esfuerzo de scraping/integración. -->

## No alcance (qué explícitamente no incluye esta versión)

- Compra o checkout dentro de la plataforma (AhorraPE solo compara y redirige a la tienda original).
- Comparación de precios fuera de tiendas online peruanas (marketplaces internacionales, tiendas
  físicas, redes sociales/informales).
- Alertas automáticas de bajada de precio por notificación push/email/WhatsApp.
<!-- REVISAR: esto puede ser una funcionalidad deseable a futuro, se excluye de esta primera versión por decisión de alcance, no por falta de valor. -->
- Cuentas de usuario con perfiles, listas de deseos o historial personalizado de búsquedas.
- Reseñas u opiniones de usuarios sobre productos o tiendas.
- Comparación de disponibilidad de stock en tiempo real garantizada (se muestra el último precio
  capturado, no se garantiza disponibilidad instantánea).

## Criterios de éxito

- Un usuario puede ir desde la pantalla principal hasta ver el precio comparado de un producto en
  al menos 2 tiendas distintas en 3 clics o menos.
- El histórico de precios de un producto muestra datos de al menos las últimas 4 semanas (una vez
  que el producto lleva ese tiempo siendo trackeado).
- La "mejor oferta vigente" mostrada corresponde al precio más bajo capturado en las últimas 24
  horas entre las tiendas comparadas para ese producto.
<!-- REVISAR: definir la frecuencia real de actualización de precios (cada cuántas horas se vuelve a capturar el precio por tienda), ya que condiciona qué tan "vigente" es realmente la oferta mostrada. -->
- El buscador central devuelve resultados relevantes para al menos el 90% de búsquedas de productos
  que sí existen en el catálogo (medido con un set de búsquedas de prueba).

## Casos borde a contemplar

- El producto buscado no existe en el catálogo o no fue encontrado en ninguna tienda: mostrar
  estado vacío claro, no un buscador "roto" o sin resultados sin explicación.
- El producto solo está disponible en una tienda (no hay comparación posible): mostrar el precio
  de esa única tienda sin simular una comparación inexistente.
- Una tienda deja de responder o el precio no pudo actualizarse (dato desactualizado o caído):
  indicar la fecha/hora de la última actualización de precio en vez de mostrar un dato silenciosamente
  obsoleto como si fuera vigente.
- El mismo producto aparece con nombres/variantes ligeramente distintos entre tiendas (ej. distinto
  modelo, color o capacidad): evitar comparar productos que en realidad no son equivalentes.
<!-- REVISAR: este es un riesgo técnico central (matching de productos entre tiendas) y probablemente merece su propia investigación antes de diseño. -->
- Precio con error evidente de captura (ej. precio en 0 o absurdamente bajo/alto por error de
  scraping): no debe mostrarse como "mejor oferta vigente" sin validación mínima.
- Picos de tráfico durante eventos tipo CyberWow, cuando más usuarios comparan precios a la vez.

## Supuestos y riesgos abiertos

- Se asume que existe una forma viable (scraping, API pública o acuerdo) de obtener precios de las
  tiendas online peruanas objetivo; esto no está confirmado y es el mayor riesgo técnico del proyecto.
<!-- REVISAR: definir método de obtención de datos por tienda (scraping propio, API oficial, feed de terceros) — impacta legalidad, mantenimiento y frecuencia de actualización. -->
- Se asume que "tiendas online de Perú" se refiere en esta primera versión a un conjunto acotado de
  retailers grandes (ej. tipo Falabella, Ripley, Plaza Vea, etc.), no a todo el comercio electrónico
  peruano.
<!-- REVISAR: confirmar la lista de tiendas objetivo con el usuario antes de iniciar diseño/arquitectura. -->
- Se asume que el proyecto es un proyecto académico/final de curso (DMC), lo cual puede acotar el
  alcance técnico real (número de tiendas, infraestructura, frecuencia de actualización) respecto a
  lo descrito aquí como visión de producto completa.
<!-- REVISAR: confirmar si este PRD debe reflejar la versión "producto completo" o la versión "entregable de curso", ya que puede cambiar significativamente el alcance. -->
- Riesgo legal/de términos de servicio al extraer precios de tiendas de terceros vía scraping, si
  ese es el método elegido para obtener los datos.
