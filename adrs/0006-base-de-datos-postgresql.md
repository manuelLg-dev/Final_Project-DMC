# ADR 0006: PostgreSQL como base de datos compartida

## Estado

Aceptado

## Contexto

En la arquitectura del ADR-0002 la base de datos es el contrato entre los dos componentes: la
ingesta escribe (semilla + scraper) y la aplicación web lee. El modelo del ADR-0003 incluye una
serie temporal (`CapturaPrecio`) sobre la que se calculan agregaciones no triviales: mejor oferta
vigente (mínimo en 24 h por producto), historial de 4+ semanas, mayores caídas de precio e índice
de precios por categoría para el dashboard. Debe soportar escrituras de jobs de ingesta
concurrentes con lecturas de la web.

## Decisión

PostgreSQL como única base de datos del sistema, ejecutada en Docker para desarrollo local y en
un servicio gestionado con capa gratuita (ej. Supabase o Neon) si el proyecto se despliega.

## Alternativas consideradas

- **SQLite** — Cero infraestructura (un archivo), atractivo para una demo local, pero limita la
  concurrencia entre los jobs de ingesta y la web (escritor único), y complica cualquier deploy
  donde webapp e ingesta no compartan sistema de archivos — exactamente la topología del ADR-0002.
- **MySQL/MariaDB** — Relacional igualmente válido, pero sin ventaja técnica sobre PostgreSQL
  para este proyecto y con un ecosistema de servicios gratuitos y tooling TypeScript algo menos
  conveniente; solo se justificaría por familiaridad previa, que no fue el caso.

## Consecuencias

- Las agregaciones del dashboard y la "mejor oferta vigente" se resuelven con SQL estándar
  (funciones de ventana, `MIN` sobre rangos temporales) sin capa adicional.
- Ingesta y webapp pueden correr en procesos/máquinas distintas apuntando a la misma DB, como
  exige el ADR-0002.
- **Trade-off:** requiere infraestructura real (contenedor Docker local o servicio externo) que
  hay que instalar, versionar (migraciones) y mantener corriendo para cualquier demo — más pesado
  que el archivo único de SQLite.
