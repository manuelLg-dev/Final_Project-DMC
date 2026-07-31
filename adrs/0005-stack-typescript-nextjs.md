# ADR 0005: Stack TypeScript end-to-end (Next.js + scripts Node para ingesta)

## Estado

Aceptado

## Contexto

Hay que elegir lenguaje y framework para los dos componentes del ADR-0002: la aplicación web
(frontend + API REST del ADR-0004, con pantallas interactivas como el gráfico de historial y los
filtros de resultados) y el pipeline de ingesta (generador de semilla + scraper del ADR-0001).
Restricciones: equipo de una persona y plazo de entrega de curso — cada lenguaje adicional
multiplica tooling, dependencias y contexto que mantener.

## Decisión

Todo el proyecto se implementa en TypeScript:

- **Aplicación web:** Next.js — frontend React y API REST (route handlers `/api/*`) en un solo
  deploy, encajando exactamente con el componente "webapp full-stack" del ADR-0002.
- **Pipeline de ingesta:** scripts Node/TypeScript ejecutables por CLI o job programado — el
  generador de semilla y el scraper (Playwright y/o Cheerio) escribiendo por el módulo de ingesta
  común.

## Alternativas consideradas

- **Python en todo (FastAPI + frontend con templates o React/Vite)** — El mejor ecosistema de
  scraping y un solo lenguaje, pero el frontend rico que exige el DESIGN.md (gráfico interactivo,
  filtros, estados resaltados) es menos natural: con templates queda austero y con React/Vite se
  terminan manteniendo dos ecosistemas de todos modos.
- **Mixto (Next.js + ingesta en Python)** — Cada componente en su mejor ecosistema con la DB como
  único acoplamiento, pero duplica tooling, entornos y contexto para una sola persona; el ahorro
  del ecosistema Python de scraping no compensa el costo de mantener dos lenguajes en un plazo de
  curso.

## Consecuencias

- Un solo lenguaje, un solo gestor de dependencias y tipos compartibles entre la ingesta, la API
  y el frontend (mitiga el riesgo de desincronización de contrato señalado en ADR-0004).
- Next.js resuelve de fábrica la combinación frontend + API en un deploy que pide el ADR-0002.
- **Trade-off:** el ecosistema de scraping en Node (Playwright/Cheerio) es más limitado que el de
  Python; si el scraper best-effort se topa con protecciones anti-bot duras, habrá menos
  herramientas maduras disponibles. Aceptable porque el ADR-0001 ya acota el scraper a 1-2
  tiendas accesibles y la demo no depende de él.
