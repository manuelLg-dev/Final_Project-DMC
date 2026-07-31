# ADR 0002: Dos componentes — aplicación web full-stack y pipeline de ingesta

## Estado

Aceptado

## Contexto

El DESIGN.md define una web desktop con cinco pantallas (home/buscador, resultados de búsqueda,
detalle de producto con historial, dashboard de tendencias y catálogo por categorías). El ADR-0001
añade un pipeline de ingesta con dos productores (generador de semilla y scraper best-effort) que
deben escribir por un contrato común. Hay que decidir en cuántos componentes se organiza el
sistema, sabiendo que lo construye y opera una sola persona con plazo de curso.

## Decisión

El sistema se organiza en dos componentes dentro de un mismo repositorio:

1. **Aplicación web** (frontend + API en un solo deploy): sirve la interfaz y expone los datos de
   búsqueda, comparación, historial y tendencias leyendo de la base de datos.
2. **Pipeline de ingesta** (proceso independiente): el generador de semilla y el scraper escriben
   precios y productos a la misma base de datos a través de un contrato de ingesta común,
   ejecutados como jobs programados o manuales.

Ambos componentes se comunican únicamente a través de la base de datos: la ingesta escribe, la
aplicación web lee.

## Alternativas consideradas

- **Monolito único (web + ingesta en el mismo proceso)** — La opción más simple de levantar, pero
  acopla el ciclo de vida del scraper al de la web: un job colgado o un scraper roto puede afectar
  la aplicación, y el pipeline deja de poder mostrarse/ejecutarse como pieza independiente.
- **Tres componentes (SPA + API + ingesta)** — Separación máxima y cada pieza evaluable por
  separado, pero multiplica la configuración y coordinación (dos servidores web + jobs + DB) sin
  que el proyecto de curso tenga un segundo cliente que justifique una API independiente.

## Consecuencias

- La ingesta corre y falla de forma aislada: un scraper caído nunca tumba la web, alineado con el
  caso borde del PRD ("una tienda deja de responder... indicar la fecha/hora de la última
  actualización").
- Solo hay dos piezas que operar (más la base de datos), acorde al equipo de una persona.
- **Trade-off:** frontend y API comparten deploy; si a futuro aparece otro cliente (ej. móvil,
  mencionado en el DESIGN.md como pendiente), habrá que extraer la API como servicio propio.
- La base de datos se convierte en el contrato entre componentes: cualquier cambio de esquema
  afecta a ambos y debe versionarse con cuidado (migraciones).
