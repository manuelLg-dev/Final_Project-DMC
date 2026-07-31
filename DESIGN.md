# AhorraPE — Documentación de Wireframes

## 1. Contexto
Plataforma de comparación de precios (Perú). Usuario objetivo: comprador puntual que busca la mejor oferta hoy entre tiendas (Falabella, Ripley, Plaza Vea, Oechsle). Plataforma: web desktop.

## 2. Proceso seguido
1. Leí el PRD del repo para extraer objetivos, usuarios y funcionalidades clave.
2. Hice preguntas de alcance antes de bocetar: qué pantallas priorizar, plataforma, tiendas de ejemplo, foco de usuario, cómo destacar "mejor oferta", nivel de estados vacíos, y grado de variación entre layouts.
3. Con las respuestas, definí un sistema de wireframe único (mismo header/footer en todas las pantallas) y generé variaciones sutiles de layout dentro de esa estructura, no conceptos radicalmente distintos.
4. Cada tanda de pantallas se agrupó como un "turno" con opciones etiquetadas (1a, 1b… / 2a, 2b…) para poder referenciarlas y compararlas lado a lado.
5. Completé el set de pantallas faltantes identificadas contra el PRD (resultados, detalle, tendencias, catálogo), reutilizando el mismo header/footer.

## 3. Sistema de wireframe (reglas fijas en todas las pantallas)
- **Header:** logo "AhorraPE" + nav (Categorías, Tendencias, Cómo funciona) + buscador.
- **Footer:** copyright + disclaimer "Precios referenciales, verifica en tienda".
- **Mejor oferta:** se resalta con fondo amarillo suave + borde dorado + tag "Mejor precio · Tienda", aplicado a la fila/tarjeta completa (no solo un badge).
- **Placeholders:** cajas con borde punteado para fotos/gráficos aún sin definir.
- Estilo deliberadamente "boceto" (fuente manuscrita, boxes) para dejar claro que es etapa de wireframe, no visual final.

## 4. Pantallas generadas y su decisión de layout

### 1 — Home / Buscador central (3 variantes)
- **1a** Buscador centrado + resultados en tarjetas: prioriza descubrimiento con tags de categoría rápida.
- **1b** Hero oscuro + comparación directa por tienda (sin cards intermedias): va directo al listado de tiendas por producto, más rápido para el comprador puntual.
- **1c** Sidebar de categorías + grilla + estado vacío visible: para navegación por categoría además de búsqueda.

### 2 — Pantallas faltantes según el PRD
- **2a** Resultados de búsqueda: filtros por tienda/precio/orden + lista con la mejor oferta resaltada.
- **2b** Detalle de producto: comparación por tienda + historial de precios (gráfico) + alerta de precio.
- **2c** Dashboard de tendencias: mayores caídas de precio, más buscados, índice de precios por categoría.
- **2d** Categorías / catálogo: grilla de categorías como entrada alternativa a la búsqueda.

## 5. Pendiente / próximos pasos sugeridos
- Login / registro. El botón "Ingresar" se retiró de los wireframes de v1 por la misma razón que
  la alerta de precio: el PRD no contempla cuentas de usuario en este alcance (ver ADR-0003 y
  TECH-DESIGN.md). Un sistema de login/registro habilitaría a futuro: alertas de precio
  personalizadas, favoritos / lista de seguimiento de productos, e historial de búsquedas
  personalizado.
- Estado "producto no encontrado" a nivel de página completa.
- Estado "solo 1 tienda disponible" en detalle de producto.
- Estado "precio desactualizado" (badge de antigüedad de dato).
- Definir flujo mobile si el foco escala a esa plataforma.

## 6. Cómo regenerar este proceso
1. Releer el PRD y extraer: usuarios, features must-have, tono de marca.
2. Preguntar antes de bocetar: alcance de pantallas, plataforma, ejemplos reales (tiendas/marcas), foco de usuario, tratamiento visual de datos clave (ej. mejor oferta), nivel de casos borde, y grado de variación deseado.
3. Fijar un sistema (header/footer/estados) antes de multiplicar pantallas.
4. Generar variantes sutiles dentro de ese sistema, agrupadas por turno con ids referenciables.
5. Cruzar contra el PRD para detectar pantallas faltantes y completarlas con el mismo sistema.
6. Documentar decisiones en este archivo a medida que se agregan pantallas.
