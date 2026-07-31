# ADR 0011: Acceso a datos y migraciones con Drizzle ORM

## Estado

Aceptado (2026-07-31)

## Contexto

Dos decisiones previas apuntan a una herramienta que ninguna llegó a elegir. El ADR-0002
establece que "la base de datos se convierte en el contrato entre componentes: cualquier cambio de
esquema afecta a ambos y debe versionarse con cuidado (migraciones)". El ADR-0005 justifica en
parte el stack TypeScript por los "tipos compartibles entre la ingesta, la API y el frontend
(mitiga el riesgo de desincronización de contrato señalado en ADR-0004)". Ninguno decide **cómo**
se define el esquema, cómo se versiona ni de dónde salen esos tipos (revisión adversarial,
sugerencia S1).

Restricciones concretas que condicionan la elección:

- **Dos consumidores del mismo esquema:** el pipeline de ingesta escribe, la webapp lee (ADR-0002).
- **Consultas no triviales en SQL:** el ADR-0006 eligió PostgreSQL precisamente por las
  agregaciones del dashboard (mínimo por ventana de 24 h, caídas en 7 días, índice por categoría
  con canasta fija) — funciones de ventana y agregados que ningún ORM expresa cómodamente.
- **Features de PostgreSQL que el esquema necesita:** el ADR-0009 exige una columna generada
  `tsvector`, índices GIN y las extensiones `unaccent` y `pg_trgm`.
- Equipo de una persona y plazo de curso.

## Decisión

**Drizzle ORM** como capa de acceso a datos y **Drizzle Kit** para las migraciones:

- El esquema se declara en TypeScript (`src/db/schema.ts`), como fuente única para ambos
  componentes. Los tipos de fila se derivan de esa declaración (`$inferSelect`/`$inferInsert`) y
  son los que viajan hacia la API y el frontend, cumpliendo el "tipos compartibles" del ADR-0005.
- Las migraciones se generan con `drizzle-kit generate` y quedan **versionadas como archivos SQL
  en el repositorio**, aplicadas con `drizzle-kit migrate`.
- Las consultas simples (catálogo, detalle, listados) usan el query builder; las agregaciones del
  dashboard y la búsqueda del ADR-0009 se escriben en **SQL crudo** mediante el operador `sql` de
  Drizzle, que conserva el tipado del resultado.

## Alternativas consideradas

- **Prisma** — La opción más cómoda del ecosistema: mejor DX, migraciones maduras y un cliente
  generado excelente. Descartada por fricción concreta con decisiones ya tomadas, no por gusto: su
  esquema propio (`schema.prisma`) no expresa columnas generadas `tsvector` ni índices GIN, así
  que el ADR-0009 obligaría a migraciones SQL escritas a mano fuera del modelo —quedándose el
  esquema declarado sin reflejar el esquema real—, y todas las agregaciones del dashboard caerían
  en `$queryRaw`, que devuelve resultados sin tipar. Se pagaría el peso de un ORM completo
  usándolo por fuera justo donde más importa. Añade además un motor de query y un paso de
  generación al pipeline de build.
- **SQL puro + `node-pg-migrate`** — Control total, cero abstracción, y todo lo del ADR-0009 y del
  dashboard se escribe de forma natural. Descartada porque deja sin resolver exactamente aquello
  para lo que el ADR-0005 eligió TypeScript: los tipos de las filas se escriben y se mantienen a
  mano en los dos componentes, y nada obliga a que sigan correspondiendo al esquema real. Es el
  riesgo de desincronización del ADR-0004 reaparecido en la capa de datos.

## Consecuencias

- Existe **una sola declaración del esquema** para ingesta y webapp, y los tipos salen de ella:
  el "contrato compartido" del ADR-0002 y del ADR-0005 pasa de intención a mecanismo.
- El SQL difícil se escribe como SQL. Las agregaciones del dashboard y el full-text del ADR-0009
  no pelean contra la abstracción, que era el riesgo real de meter un ORM en este proyecto.
- Las migraciones son archivos SQL en git: revisables, y aplicables tanto en local (Docker) como
  en el gestionado (ADR-0006) y desde el workflow de ingesta (ADR-0010).
- **Trade-off:** Drizzle es menos maduro que Prisma y su tooling de migraciones es más áspero —en
  particular, los cambios destructivos o los renombrados requieren revisar el SQL generado antes
  de aplicarlo. Aceptable en un proyecto que aún no tiene datos de producción que perder.
- **Trade-off:** al escribir las agregaciones en SQL crudo, el tipo del resultado lo declara quien
  escribe la consulta; Drizzle lo transporta pero no lo verifica contra la base. El tipado protege
  el esquema, no cada `SELECT` complejo.
- La columna generada del ADR-0009 y las extensiones (`unaccent`, `pg_trgm`) se declaran en la
  migración inicial, que es donde el ADR-0009 ya pedía habilitarlas.