# Index Scrollytelling · mcp-oracle-database

**Fecha:** 2026-08-12
**Estado:** Aprobado por el usuario (concepto + storyboard)
**Alcance:** Reescritura completa de `src/pages/index.astro`

## Objetivo

Convertir el index actual (vitrina estática de 3 informes) en una página de
presentación de proyecto estilo Apple: **scrollytelling** con animaciones GSAP
ScrollTrigger que narra el proyecto `mcp-oracle-database` acto por acto.

Fuente de contenido: README oficial de https://github.com/zademy/mcp-oracle-database.

## Restricciones duras

- **NO modificar** `astro.config.mjs` ni ningún archivo de configuración de
  Astro. El despliegue sigue igual (workflow `.github/workflows/deploy.yml`).
- **NO modificar** los informes en `public/informes/`. Sus enlaces y datos
  (87×, 25×, 284 registros, 0 diferencias) se conservan tal cual.
- La página debe funcionar servida bajo el `BASE_URL` de GitHub Pages
  (ya se usa `import.meta.env.BASE_URL`).
- Tema claro/oscuro existente se conserva (script inline + `data-theme`).

## Tecnología

- **GSAP + ScrollTrigger desde CDN** (`cdn.jsdelivr.net/npm/gsap@3/dist/...`),
  cargados con `<script src>` al final del body. Sin dependencias npm nuevas.
- Scripts de animación inline (`is:inline`) en el propio `index.astro`.
- Todo el CSS sigue inline en el componente (patrón actual del archivo).
- **Accesibilidad:** si `prefers-reduced-motion: reduce`, GSAP se desactiva
  (`ScrollTrigger.disable()` / `gsap.matchMedia`) y todo el contenido queda
  visible sin animación.

## Storyboard — 6 actos

### Acto 1 · Hero

- Eyebrow "Proyecto de innovación · Model Context Protocol" (se conserva).
- H1: presentar el MCP server (ej. "Oracle para tu IA, con candado").
- Lede breve: servidor MCP STDIO, Spring Boot, least-privilege.
- CTAs: "Repositorio MCP ↗" (GitHub) y "Ver la evidencia ↓" (ancla al acto 5).
- Terminal a la derecha con **animación de tipeo GSAP**: reproduce una sesión
  (`test_connection` → `list_tables` → `describe_table`) línea a línea al
  entrar en viewport.

### Acto 2 · El problema (sección pinned)

- Texto fijo a un lado: "¿Y si la IA ejecuta DDL destructivo?"
- Al scrollear, la terminal muestra el intento `DROP TABLE ...` y la respuesta
  de Oracle: **ORA-01031: insufficient privileges**.
- Implementación: contenedor con `pin: true` y scrub sobre la secuencia
  (2-3 pasos de texto que aparecen según el avance del scroll).

### Acto 3 · La solución (modelo de seguridad)

- Diagrama animado: `Cliente MCP → mcp-oracle-db (Spring Boot · STDIO) →
Oracle (usuario least-privilege = la única barrera)`. Cajas + flechas en
  HTML/CSS que entran escalonadas con GSAP.
- Counters animados al entrar en viewport: **70** herramientas, **0** puertos
  web (STDIO-only), **1** sola barrera (Oracle).
- Bullets: sin DDL/DCL posible, credenciales por variables de entorno,
  Java 26 virtual threads.

### Acto 4 · Las herramientas (scrub)

- Sección con `pin` + `scrub`: al scrollear desfilan las 8 categorías con
  herramientas reales del README:
  1. Introspección de esquema (`list_tables`, `describe_table`, `get_ddl`)
  2. Diagnóstico de rendimiento (`list_top_sql`, `list_blocked_sessions`, `get_wait_events`)
  3. Advisory de índices (`suggest_index`, `run_sql_tuning_advisor`)
  4. Helpers de datos (`find_duplicates`, `validate_fk_integrity`, `find_free_id`)
  5. SQL y DML (`run_query`, `execute_dml_preview`, `explain_plan`)
  6. Grafos de esquema (`get_fk_graph`, `get_dependency_graph`)
  7. PL/SQL (`call_procedure`, `get_plsql_errors`)
  8. Sistema (`test_connection`, `oracle_mcp_health_report`)
- Cada categoría: tarjeta con título + 2-3 nombres de herramientas en mono.
- Números exactos del README: 70 tools, 4 resources URI, 6 prompts,
  13 completions (se muestran como contadores).

### Acto 5 · La evidencia

- Los 3 informes actuales (IMSS, ISSSTE, Matrimonio) con las mismas tarjetas
  de métricas existentes, más **counters GSAP** en los números clave:
  87×, 284 registros, 0 diferencias.
- Enlaces a `informes/*.html` sin cambios.

### Acto 6 · Cierre (quickstart + CTA)

- Bloque JSON de configuración MCP (el del README paso 4) con botón "Copiar"
  (`navigator.clipboard`).
- CTA final grande al repositorio oficial.
- Footer actual (crédito mcp-oracle-database, nota de filtros de negocio).

## Estructura del archivo

Un solo archivo: `src/pages/index.astro` (patrón actual, ~1200-1400 líneas
estimadas tras la reescritura). Secciones:

1. Frontmatter: datos de informes (conservados), categorías de herramientas,
   textos de actos.
2. `<style>`: variables de tema existentes + nuevos estilos por acto.
3. Markup: `<section data-act="1..6">` para cada acto.
4. Scripts: toggle de tema (existente), carga GSAP desde CDN, timeline de
   animaciones con guardas `prefers-reduced-motion`.

## Verificación

1. `npm run build` — build de Astro sin errores.
2. `astro dev` + navegación: cada acto anima al entrar en viewport, tema
   claro/oscuro funciona, enlaces a informes resuelven con BASE_URL.
3. Comprobar `prefers-reduced-motion` (emular en DevTools): la página se lee
   completa sin animaciones.
4. Confirmar que `astro.config.mjs` y `public/informes/` quedan intactos
   (`git status`).

## Riesgos

- **CDN de GSAP en Pages:** GitHub Pages sirve sobre HTTPS; jsdelivr es
  confiable. Si falla la carga, la página debe seguir legible (el contenido
  nunca debe quedar oculto esperando animación: ocultar solo con JS activo).
- **Pin/scrub en móvil:** GSAP lo soporta, pero verificar en viewport angosto.
