# Index Scrollytelling · Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Reescribir `src/pages/index.astro` como página scrollytelling estilo Apple que presenta el proyecto mcp-oracle-database en 6 actos animados con GSAP ScrollTrigger.

**Architecture:** Un solo archivo Astro (patrón actual del repo): frontmatter con datos, `<style>` inline con variables de tema claro/oscuro, markup por secciones `<section class="act ...">`, y un bloque `<script is:inline>` final que registra GSAP + ScrollTrigger (cargados desde CDN) y arma los timelines dentro de `gsap.matchMedia()` respetando `prefers-reduced-motion`.

**Tech Stack:** Astro 7 (estático), CSS inline, GSAP 3 + ScrollTrigger vía CDN jsdelivr, JS inline sin dependencias npm.

## Global Constraints

- NO modificar `astro.config.mjs` ni ningún archivo de configuración de Astro.
- NO modificar `public/informes/*`. Los datos/enlaces de los 3 informes se conservan verbatim.
- Enlaces internos usan `${base}` (`import.meta.env.BASE_URL`).
- GSAP desde CDN con versión fijada y SRI (hashes oficiales de jsdelivr para gsap@3.13.0):
  - `https://cdn.jsdelivr.net/npm/gsap@3.13.0/dist/gsap.min.js` — `integrity="sha384-lsAbgfRKMpDitFMvVeLJU0sq3EMnOhnzdWsstB8P0LY="`
  - `https://cdn.jsdelivr.net/npm/gsap@3.13.0/dist/ScrollTrigger.min.js` — `integrity="sha384-MIIZOQ5eO4TNoMSB5wyqmCCIOuEL2kTm6aFJqBqsSz8="`
    Ambos con `crossorigin="anonymous"`. Sin dependencias npm nuevas.
- Si GSAP no carga o hay `prefers-reduced-motion: reduce`, la página debe leerse completa: ningún contenido queda oculto sin JS activo (se oculta solo vía `gsap.set`/timelines dentro de `matchMedia`).
- Shell Windows: si `git commit -m "texto con espacios"` falla por quoting, escribir el mensaje en `.superpowers/sdd/msg-<task>.txt` y commitear con `git commit -F <archivo>`.
- Verificación estándar tras cada tarea: `npm run build` debe terminar sin errores.

---

### Task 1: Esqueleto nuevo + Acto 1 (Hero con terminal tipeada)

**Files:**

- Rewrite: `src/pages/index.astro` (790 líneas actuales → nuevo archivo)

**Interfaces:**

- Produce para tareas posteriores: estructura `<head>` completa (tema, variables CSS), `<header class="site">`, `<section class="act act-hero">`, `<footer class="site">`, y el bloque final de scripts (theme toggle + GSAP CDN + IIFE `window.__initActs`). Las tareas 2-6 insertan actos adicionales antes de `</main>` y añaden su inicialización dentro de la IIFE.

- [ ] **Step 1: Reemplazar `src/pages/index.astro` con el nuevo esqueleto**

Estructura completa del archivo (frontmatter + head + hero + footer + scripts). El CSS incluye TODAS las variables de tema y utilidades base; cada tarea posterior añade sus propios estilos:

```astro
---
const base = import.meta.env.BASE_URL;

const reports = [
  {
    chip: "Q-01",
    badge: "IMSS",
    title: "Desempleo IMSS · SIRH CARE",
    description:
      "La consulta original repetía la misma búsqueda de folio padre 4 veces por fila. La optimizada pregunta una sola vez y trabaja con 4 procesos en paralelo.",
    href: `${base}informes/informe-comparativa-query-01.html`,
    highlight: { value: "87×", label: "menor costo de joins" },
    metrics: [
      { label: "Costo de joins", before: "41,055", after: "474" },
      { label: "Paralelismo", before: "1", after: "4" },
      { label: "Registros verificados", after: "110" },
      { label: "Diferencias", after: "0" },
    ],
  },
  {
    chip: "Q-01",
    badge: "ISSSTE",
    title: "Desempleo ISSSTE · S6",
    description:
      "El plan original bajaba 2 niveles por cada fila. La versión optimizada elimina las búsquedas por fila y activa paralelismo ×4 confirmado por Oracle.",
    href: `${base}informes/informe-comparativa-query-01-issste.html`,
    highlight: { value: "25×", label: "menor costo de joins" },
    metrics: [
      { label: "Costo de joins", before: "10,637", after: "431" },
      { label: "Búsquedas por fila", before: "2 niveles", after: "0" },
      { label: "Registros verificados", after: "15" },
      { label: "Paralelismo", after: "×4" },
    ],
  },
  {
    chip: "Q-01",
    badge: "MATRIMONIO",
    title: "Matrimonio · S18",
    description:
      "Cuatro búsquedas repetidas por fila en la versión original. La optimizada las reduce a cero con un cruce inteligente de listas y paralelismo ×4.",
    href: `${base}informes/informe-comparativa-query-01-matrimonio.html`,
    highlight: { value: "87×", label: "menor costo de joins" },
    metrics: [
      { label: "Costo de joins", before: "41,305", after: "473" },
      { label: "Búsquedas repetidas", before: "4", after: "0" },
      { label: "Registros verificados", after: "159" },
      { label: "Paralelismo", after: "×4" },
    ],
  },
];

const glance = [
  { value: "3", label: "informes comparativos" },
  { value: "284", label: "registros verificados" },
  { value: "0", label: "diferencias vs original" },
  { value: "87×", label: "reducción máxima de costo" },
];

const toolCategories = [
  { name: "Introspección de esquema", tools: ["list_tables", "describe_table", "get_ddl"] },
  { name: "Diagnóstico de rendimiento", tools: ["list_top_sql", "list_blocked_sessions", "get_wait_events"] },
  { name: "Advisory de índices", tools: ["suggest_index", "run_sql_tuning_advisor", "list_unused_indexes"] },
  { name: "Helpers de datos", tools: ["find_duplicates", "validate_fk_integrity", "find_free_id"] },
  { name: "SQL y DML", tools: ["run_query", "execute_dml_preview", "explain_plan"] },
  { name: "Grafos de esquema", tools: ["get_fk_graph", "get_dependency_graph"] },
  { name: "PL/SQL", tools: ["call_procedure", "get_plsql_errors", "describe_plsql"] },
  { name: "Sistema", tools: ["test_connection", "oracle_mcp_health_report"] },
];

const capabilityNumbers = [
  { value: 70, suffix: "", label: "herramientas MCP" },
  { value: 4, suffix: "", label: "recursos URI" },
  { value: 6, suffix: "", label: "prompts guiados" },
  { value: 13, suffix: "", label: "completions" },
];
---

<!doctype html>
<html lang="es-MX">
  <head>
    <meta charset="UTF-8" />
    <meta name="viewport" content="width=device-width, initial-scale=1.0" />
    <meta
      name="description"
      content="mcp-oracle-db: servidor MCP para Oracle Database. Introspección de esquemas, SQL seguro y diagnóstico de rendimiento, con seguridad least-privilege impuesta por Oracle."
    />
    <link rel="icon" type="image/svg+xml" href={`${base}favicon.svg`} />
    <title>mcp-oracle-db · Oracle para tu IA, con candado</title>
    <script is:inline>
      (function () {
        const stored = localStorage.getItem("theme");
        const prefersDark = window.matchMedia(
          "(prefers-color-scheme: dark)",
        ).matches;
        document.documentElement.dataset.theme =
          stored ?? (prefersDark ? "dark" : "light");
      })();
    </script>
    <style>
      /* CONSERVAR VERBATIM todo el bloque <style> del index.astro actual
         (variables de tema, utilidades, hero, terminal, stats-strip,
         cards, metrics, footer): las tareas posteriores reusan esos
         estilos. No borrar nada del CSS existente en esta tarea. */
    </style>
  </head>
  <body>
    <header class="site">
      <div class="container">
        <div class="brand"><span class="dot"></span> MCP · Oracle DB</div>
        <button id="theme-toggle" type="button" aria-label="Cambiar tema"
          >🌗 Tema</button
        >
      </div>
    </header>

    <main>
      <section class="act act-hero" id="inicio">
        <div class="container">
          <div class="hero-grid">
            <div>
              <span class="eyebrow">
                <span class="dot"></span> Proyecto de innovación · Model Context Protocol
              </span>
              <h1>
                Oracle para tu IA, <span class="grad">con candado</span>
              </h1>
              <p class="lede">
                Un servidor MCP construido con Spring Boot que deja a cualquier
                cliente de IA introspectar esquemas, ejecutar SQL y diagnosticar
                rendimiento. La seguridad no es un filtro en la app: es el propio
                Oracle, con un usuario sin privilegios DDL.
              </p>
              <div class="hero-actions">
                <a
                  class="btn-primary"
                  href="https://github.com/zademy/mcp-oracle-database"
                  rel="noopener">Repositorio MCP ↗</a
                >
                <a class="btn-ghost" href="#evidencia">Ver la evidencia ↓</a>
              </div>
            </div>

            <div class="terminal hero-terminal" aria-hidden="true">
              <div class="terminal-bar">
                <span></span><span></span><span></span>
                <em>mcp-oracle-db · primera sesión</em>
              </div>
              <pre><span class="t-line">› <span class="t-tool">test_connection</span>()
  <span class="t-ok">✔ Oracle 19c</span> · usuario mcp_user · esquema MCP_USER</span>
<span class="t-line">› <span class="t-tool">list_tables</span>(schema="HR")
  <span class="t-num">14</span> tablas visibles</span>
<span class="t-line">› <span class="t-tool">describe_table</span>(schema="HR", table="EMPLOYEES")
  <span class="t-num">11</span> columnas · PK EMP_EMP_ID_PK</span></pre>
            </div>
          </div>

          <div class="stats-strip">
            {
              glance.map((s) => (
                <div class="stat">
                  <b class="mono">{s.value}</b>
                  <span>{s.label}</span>
                </div>
              ))
            }
          </div>
        </div>
      </section>
    </main>

    <footer class="site">
      <div class="container">
        Generado con Astro ·
        <a href="https://github.com/zademy/mcp-oracle-database" rel="noopener"
          >mcp-oracle-database</a
        >
        · Licencia MIT © Sadot Hdz. Moreno
      </div>
    </footer>

    <script is:inline>
      document.getElementById("theme-toggle").addEventListener("click", () => {
        const current =
          document.documentElement.dataset.theme === "dark" ? "dark" : "light";
        const next = current === "dark" ? "light" : "dark";
        document.documentElement.dataset.theme = next;
        localStorage.setItem("theme", next);
      });
    </script>
    <script
      src="https://cdn.jsdelivr.net/npm/gsap@3.13.0/dist/gsap.min.js"
      integrity="sha384-lsAbgfRKMpDitFMvVeLJU0sq3EMnOhnzdWsstB8P0LY="
      crossorigin="anonymous"></script>
    <script
      src="https://cdn.jsdelivr.net/npm/gsap@3.13.0/dist/ScrollTrigger.min.js"
      integrity="sha384-MIIZOQ5eO4TNoMSB5wyqmCCIOuEL2kTm6aFJqBqsSz8="
      crossorigin="anonymous"></script>
    <script is:inline>
      (function () {
        if (!window.gsap || !window.ScrollTrigger) return;
        gsap.registerPlugin(ScrollTrigger);

        const mm = gsap.matchMedia();
        mm.add("(prefers-reduced-motion: no-preference)", () => {
          // Acto 1: las líneas de la terminal aparecen una a una
          const lines = gsap.utils.toArray(".hero-terminal .t-line");
          gsap.set(lines, { autoAlpha: 0, y: 8 });
          gsap.to(lines, {
            autoAlpha: 1,
            y: 0,
            duration: 0.45,
            stagger: 0.55,
            delay: 0.4,
            ease: "power2.out",
          });
        });
      })();
    </script>
  </body>
</html>
```

Nota: los estilos del hero (`.hero`, `.hero::before/::after`, `.hero-grid`,
`.eyebrow`, `h1`, `.grad`, `.lede`, `.hero-actions`, `.btn-primary`,
`.btn-ghost`, `.terminal` y variantes `t-*`, `.stats-strip`) ya existen en el
bloque `<style>` conservado: añadir reglas de alias que apunten `.act-hero`
a los mismos estilos, p. ej.:

```css
.act-hero {
  position: relative;
  padding: 4.5rem 0 3rem;
}
```

(replicando el bloque `.hero` completo con el selector `.act-hero`, o
renombrando `.hero` a `.act-hero` dentro del CSS conservado; elegir renombrar).

- [ ] **Step 2: Verificar build**

Run: `npm run build`
Expected: build exitoso, `dist/index.html` generado.

- [ ] **Step 3: Verificar visualmente**

Run: `npx astro dev --background` y abrir http://localhost:4321 (o el puerto que reporte `npx astro dev status`).
Expected: hero visible, terminal line by line animation, tema claro/oscuro funciona.

- [ ] **Step 4: Commit**

```bash
git add src/pages/index.astro
git commit -m "feat(index): esqueleto scrollytelling con hero animado"
```

---

### Task 2: Acto 2 · El problema (sección pinned con ORA-01031)

**Files:**

- Modify: `src/pages/index.astro` (insertar acto en `<main>`, CSS en `<style>`, timeline en la IIFE)

**Interfaces:**

- Consume: IIFE GSAP de Task 1 (bloque `mm.add(...)`).
- Produce: `.act-problem` con `.problem-step` ocultos solo vía GSAP.

- [ ] **Step 1: Insertar el markup del acto antes de `</main>`**

```html
<section class="act act-problem" id="problema">
  <div class="container problem-grid">
    <div class="problem-copy">
      <h2 class="act-title">
        ¿Y si la IA ejecuta<br /><span class="grad">algo destructivo?</span>
      </h2>
      <p class="lede">
        Un LLM puede proponer cualquier SQL. Este servidor no intenta filtrarlo
        en la aplicación: deja que Oracle decida, con un usuario que jamás
        recibió privilegios DDL ni DCL.
      </p>
    </div>
    <div class="terminal problem-terminal">
      <div class="terminal-bar">
        <span></span><span></span><span></span>
        <em>intento de DDL · sesión real</em>
      </div>
      <pre><span class="problem-step step-ai">› <span class="t-tool">execute_dml</span>("DROP TABLE HR.EMPLOYEES")</span>
<span class="problem-step step-oracle">  <span class="t-err">✘ ORA-01031: insufficient privileges</span></span>
<span class="problem-step step-note">  Oracle es la única barrera — sin filtros en la app.</span></pre>
    </div>
  </div>
</section>
```

- [ ] **Step 2: Añadir los estilos**

```css
.act-problem {
  padding: 4rem 0;
  min-height: 100vh;
  display: flex;
  align-items: center;
}

.problem-grid {
  display: grid;
  grid-template-columns: 1fr 1.1fr;
  gap: 3rem;
  align-items: center;
}

@media (max-width: 900px) {
  .problem-grid {
    grid-template-columns: 1fr;
  }
}

.act-title {
  margin: 0;
  font-size: clamp(1.7rem, 3.4vw, 2.6rem);
  line-height: 1.12;
  letter-spacing: -0.02em;
}

.problem-terminal pre {
  min-height: 7.5rem;
}

.terminal .t-err {
  color: #f87171;
  font-weight: 700;
}

.problem-step {
  display: block;
}
```

- [ ] **Step 3: Añadir el timeline dentro de `mm.add(...)`**

```js
// Acto 2: pinned — la sección se fija y los pasos aparecen con el scroll
const steps = gsap.utils.toArray(".problem-step");
gsap.set(steps, { autoAlpha: 0, y: 6 });
const problemTl = gsap.timeline({
  scrollTrigger: {
    trigger: ".act-problem",
    start: "top top",
    end: "+=120%",
    pin: true,
    scrub: 0.5,
  },
});
problemTl.to(steps, { autoAlpha: 1, y: 0, stagger: 0.5, ease: "none" });
```

- [ ] **Step 4: Verificar build + visual**

Run: `npm run build` (éxito). En el dev server: al hacer scroll el acto se
fija y los 3 pasos aparecen secuencialmente; el texto de error se lee en rojo.

- [ ] **Step 5: Commit**

```bash
git add src/pages/index.astro
git commit -m "feat(index): acto 2 pinned con rechazo ORA-01031"
```

---

### Task 3: Acto 3 · La solución (diagrama + counters)

**Files:**

- Modify: `src/pages/index.astro`

**Interfaces:**

- Consume: IIFE GSAP.
- Produce: `.act-solution` con `.flow-node` (diagrama) y `.count[data-count]` (contadores).

- [ ] **Step 1: Insertar el markup antes de `</main>`**

```html
<section class="act act-solution" id="solucion">
  <div class="container">
    <h2 class="act-title">
      La solución: <span class="grad">Oracle es el guardián</span>
    </h2>
    <p class="lede">
      El servidor conecta con un usuario least-privilege: CREATE SESSION,
      SELECT_CATALOG_ROLE y grants por objeto. Sin DDL, sin DCL, sin cuota de
      tablespace. Todo CREATE/ALTER/DROP muere con ORA-01031.
    </p>

    <div
      class="flow"
      role="img"
      aria-label="Cliente MCP, servidor mcp-oracle-db y Oracle conectados en cadena"
    >
      <div class="flow-node">
        <b>Cliente MCP</b>
        <span>Claude Desktop · Cursor · VS Code…</span>
      </div>
      <div class="flow-arrow" aria-hidden="true">→ JSON-RPC / STDIO →</div>
      <div class="flow-node">
        <b>mcp-oracle-db</b>
        <span>Spring Boot · Java 26 · virtual threads</span>
      </div>
      <div class="flow-arrow" aria-hidden="true">→ JDBC →</div>
      <div class="flow-node flow-node--oracle">
        <b>Oracle DB</b>
        <span>usuario least-privilege = la única barrera</span>
      </div>
    </div>

    <div class="solution-numbers">
      <div class="solution-stat">
        <b class="mono count" data-count="70">70</b
        ><span>herramientas MCP</span>
      </div>
      <div class="solution-stat">
        <b class="mono count" data-count="0">0</b
        ><span>puertos web (solo STDIO)</span>
      </div>
      <div class="solution-stat">
        <b class="mono count" data-count="1">1</b
        ><span>sola barrera: Oracle</span>
      </div>
    </div>
  </div>
</section>
```

- [ ] **Step 2: Añadir los estilos**

```css
.act-solution {
  padding: 6rem 0;
  border-top: 1px solid var(--border);
}

.act-solution .act-title {
  margin-bottom: 0.75rem;
}

.flow {
  margin-top: 2.5rem;
  display: flex;
  align-items: stretch;
  gap: 0.75rem;
  flex-wrap: wrap;
  justify-content: center;
}

.flow-node {
  flex: 1 1 200px;
  max-width: 280px;
  border: 1px solid var(--border);
  background: var(--bg-card);
  border-radius: 1rem;
  padding: 1.1rem 1.2rem;
  box-shadow: var(--shadow);
}

.flow-node b {
  display: block;
  font-size: 1rem;
}

.flow-node span {
  display: block;
  margin-top: 0.3rem;
  font-size: 0.8rem;
  color: var(--text-muted);
}

.flow-node--oracle {
  border-color: color-mix(in srgb, var(--accent) 45%, transparent);
}

.flow-arrow {
  align-self: center;
  font-size: 0.72rem;
  font-family: ui-monospace, SFMono-Regular, Menlo, Consolas, monospace;
  color: var(--text-faint);
  white-space: nowrap;
}

.solution-numbers {
  margin-top: 3rem;
  display: grid;
  grid-template-columns: repeat(3, 1fr);
  gap: 1.25rem;
}

@media (max-width: 760px) {
  .solution-numbers {
    grid-template-columns: 1fr;
  }
}

.solution-stat {
  border: 1px solid var(--border);
  background: var(--bg-inset);
  border-radius: 1rem;
  padding: 1.4rem 1rem;
  text-align: center;
}

.solution-stat b {
  display: block;
  font-size: 2.4rem;
  color: var(--accent);
}

.solution-stat span {
  font-size: 0.72rem;
  letter-spacing: 0.1em;
  text-transform: uppercase;
  color: var(--text-faint);
}
```

- [ ] **Step 3: Añadir la animación dentro de `mm.add(...)`**

```js
// Acto 3: nodos del diagrama entran escalonados + contadores
gsap.from(".flow-node, .flow-arrow", {
  autoAlpha: 0,
  y: 24,
  duration: 0.6,
  stagger: 0.12,
  ease: "power2.out",
  scrollTrigger: { trigger: ".flow", start: "top 80%" },
});

document.querySelectorAll(".count[data-count]").forEach((el) => {
  const target = parseInt(el.dataset.count, 10);
  const obj = { v: 0 };
  gsap.to(obj, {
    v: target,
    duration: 1.4,
    ease: "power1.out",
    scrollTrigger: { trigger: el, start: "top 85%" },
    onUpdate: () => {
      el.textContent = Math.round(obj.v).toLocaleString("es-MX");
    },
  });
});
```

Nota: los `.count` ya traen su valor final en el HTML (70/0/1), así que sin
JS muestran el dato correcto; el contador solo lo anima desde 0.

- [ ] **Step 4: Verificar build + visual**

Run: `npm run build` (éxito). En el dev server: el diagrama entra escalonado y
los contadores suben de 0 a 70/0/1 al hacer scroll.

- [ ] **Step 5: Commit**

```bash
git add src/pages/index.astro
git commit -m "feat(index): acto 3 con diagrama de seguridad y contadores"
```

---

### Task 4: Acto 4 · Las herramientas (scrub con categorías)

**Files:**

- Modify: `src/pages/index.astro`

**Interfaces:**

- Consume: `toolCategories` y `capabilityNumbers` del frontmatter (Task 1), IIFE GSAP.
- Produce: `.act-tools` pinned con tarjetas `.tool-card`.

- [ ] **Step 1: Insertar el markup antes de `</main>`**

```html
<section class="act act-tools" id="herramientas">
  <div class="container">
    <div class="tools-intro">
      <h2 class="act-title">
        70 herramientas,<br /><span class="grad">ocho frentes</span>
      </h2>
      <div class="capability-strip">
        { capabilityNumbers.map((c) => (
        <div class="capability">
          <b class="mono">{c.value}{c.suffix}</b>
          <span>{c.label}</span>
        </div>
        )) }
      </div>
    </div>
    <div class="tool-deck">
      { toolCategories.map((cat) => (
      <article class="tool-card">
        <h3>{cat.name}</h3>
        <ul class="mono">
          {cat.tools.map((t) =>
          <li>{t}</li>
          )}
        </ul>
      </article>
      )) }
    </div>
  </div>
</section>
```

- [ ] **Step 2: Añadir los estilos**

```css
.act-tools {
  padding: 4rem 0;
  min-height: 100vh;
  display: flex;
  align-items: center;
  border-top: 1px solid var(--border);
}

.tools-intro {
  display: flex;
  align-items: flex-end;
  justify-content: space-between;
  gap: 2rem;
  flex-wrap: wrap;
  margin-bottom: 2.5rem;
}

.capability-strip {
  display: flex;
  gap: 1.5rem;
  flex-wrap: wrap;
}

.capability b {
  display: block;
  font-size: 1.5rem;
  color: var(--accent);
}

.capability span {
  font-size: 0.7rem;
  letter-spacing: 0.08em;
  text-transform: uppercase;
  color: var(--text-faint);
}

.tool-deck {
  display: grid;
  grid-template-columns: repeat(4, 1fr);
  gap: 1rem;
}

@media (max-width: 1000px) {
  .tool-deck {
    grid-template-columns: repeat(2, 1fr);
  }
}

@media (max-width: 560px) {
  .tool-deck {
    grid-template-columns: 1fr;
  }
}

.tool-card {
  border: 1px solid var(--border);
  background: var(--bg-card);
  border-radius: 0.9rem;
  padding: 1rem 1.1rem;
  box-shadow: var(--shadow);
}

.tool-card h3 {
  margin: 0 0 0.6rem;
  font-size: 0.85rem;
  letter-spacing: 0.04em;
}

.tool-card ul {
  margin: 0;
  padding: 0;
  list-style: none;
  font-size: 0.78rem;
  color: var(--text-muted);
  line-height: 1.9;
}
```

- [ ] **Step 3: Añadir la animación dentro de `mm.add(...)`**

```js
// Acto 4: pinned — las tarjetas del deck entran por tandas con el scroll
const cards = gsap.utils.toArray(".tool-card");
gsap.set(cards, { autoAlpha: 0, y: 30 });
const toolsTl = gsap.timeline({
  scrollTrigger: {
    trigger: ".act-tools",
    start: "top top",
    end: "+=160%",
    pin: true,
    scrub: 0.5,
  },
});
toolsTl.to(cards, { autoAlpha: 1, y: 0, stagger: 0.35, ease: "none" });
```

- [ ] **Step 4: Verificar build + visual**

Run: `npm run build` (éxito). En el dev server: al scrollear la sección se
fija y las 8 tarjetas aparecen en tandas; en móvil quedan en 1-2 columnas.

- [ ] **Step 5: Commit**

```bash
git add src/pages/index.astro
git commit -m "feat(index): acto 4 scrub con catalogo de herramientas"
```

---

### Task 5: Acto 5 · La evidencia (informes + counters)

**Files:**

- Modify: `src/pages/index.astro`

**Interfaces:**

- Consume: `reports` del frontmatter (Task 1).
- Produce: `.act-evidence` con las tarjetas de informe existentes y `.count-evidence` animados.

- [ ] **Step 1: Insertar el markup antes de `</main>`**

```html
<section class="act act-evidence" id="evidencia">
  <div class="container">
    <h2 class="act-title">
      La evidencia: <span class="grad">Query 01, tres entornos</span>
    </h2>
    <p class="lede">
      El servidor en acción: cada informe compara la consulta original contra la
      optimizada — plan de ejecución, costos y verificación fila por fila.
    </p>

    <div class="evidence-strip">
      <div class="evidence-stat">
        <b class="mono"
          ><span class="count-evidence" data-count="87">87</span>×</b
        >
        <span>reducción máxima de costo de joins</span>
      </div>
      <div class="evidence-stat">
        <b class="mono count-evidence" data-count="284">284</b>
        <span>registros verificados 1 a 1</span>
      </div>
      <div class="evidence-stat">
        <b class="mono count-evidence" data-count="0">0</b>
        <span>diferencias vs original</span>
      </div>
    </div>

    <div class="cards">
      { reports.map((r) => (
      <article class="card">
        <div class="card-head">
          <div class="top">
            <span class="chip mono">{r.chip}</span>
            <span class="chip">{r.badge}</span>
          </div>
          <h3>{r.title}</h3>
          <div class="hero-number mono">
            <b>{r.highlight.value}</b>
            <span>{r.highlight.label}</span>
          </div>
        </div>
        <div class="card-body">
          <p>{r.description}</p>
          <dl class="metrics">
            {r.metrics.map((m) => (
            <div>
              <dt>{m.label}</dt>
              <dd class="mono">
                {m.before && <span class="before">{m.before} →</span>}
                <span class="after">{m.after}</span>
              </dd>
            </div>
            ))}
          </dl>
          <a class="card-cta" href="{r.href}"> Ver informe completo → </a>
        </div>
      </article>
      )) }
    </div>
  </div>
</section>
```

- [ ] **Step 2: Verificar estilos existentes y añadir los nuevos**

Los estilos `.cards`, `article.card`, `.card-head` (+ `.top`, `.chip`, `h3`,
`.hero-number`), `.card-body`, `dl.metrics`, `.card-cta` YA existen en el
bloque `<style>` (conservados desde Task 1): verificar que estén presentes,
no duplicarlos. Añadir solo:

```css
.act-evidence {
  padding: 6rem 0;
  border-top: 1px solid var(--border);
}

.evidence-strip {
  margin: 2.5rem 0 3rem;
  display: grid;
  grid-template-columns: repeat(3, 1fr);
  gap: 1.25rem;
}

@media (max-width: 760px) {
  .evidence-strip {
    grid-template-columns: 1fr;
  }
}

.evidence-stat {
  border: 1px solid var(--border);
  background: var(--bg-card);
  border-radius: 1rem;
  padding: 1.5rem 1rem;
  text-align: center;
  box-shadow: var(--shadow);
}

.evidence-stat b {
  display: block;
  font-size: 2.6rem;
  color: var(--accent);
}

.evidence-stat span {
  font-size: 0.72rem;
  letter-spacing: 0.1em;
  text-transform: uppercase;
  color: var(--text-faint);
}
```

- [ ] **Step 3: Añadir la animación dentro de `mm.add(...)`**

```js
// Acto 5: tarjetas entran escalonadas + contadores de evidencia
gsap.from(".act-evidence .card", {
  autoAlpha: 0,
  y: 40,
  duration: 0.7,
  stagger: 0.18,
  ease: "power2.out",
  scrollTrigger: { trigger: ".act-evidence .cards", start: "top 80%" },
});

document.querySelectorAll(".count-evidence").forEach((el) => {
  const target = parseInt(el.dataset.count, 10);
  const obj = { v: 0 };
  gsap.to(obj, {
    v: target,
    duration: 1.6,
    ease: "power1.out",
    scrollTrigger: { trigger: el, start: "top 88%" },
    onUpdate: () => {
      el.textContent = Math.round(obj.v).toLocaleString("es-MX");
    },
  });
});
```

- [ ] **Step 4: Verificar build + enlaces**

Run: `npm run build` (éxito). En el dev server: verificar que los 3 enlaces
"Ver informe completo" resuelven (con BASE_URL) y los contadores animan.

- [ ] **Step 5: Commit**

```bash
git add src/pages/index.astro
git commit -m "feat(index): acto 5 con informes y contadores de evidencia"
```

---

### Task 6: Acto 6 · Cierre (quickstart copiable + CTA final)

**Files:**

- Modify: `src/pages/index.astro`

**Interfaces:**

- Consume: estructura del footer existente (Task 1).
- Produce: `.act-close` con bloque de código y botón copiar funcional sin GSAP.

- [ ] **Step 1: Insertar el markup antes de `</main>`**

```html
<section class="act act-close" id="quickstart">
  <div class="container">
    <h2 class="act-title">
      Conéctalo en <span class="grad">cuatro pasos</span>
    </h2>
    <p class="lede">
      Usuario least-privilege, variables de entorno, build con Maven y este
      bloque en tu cliente MCP:
    </p>

    <div class="code-card">
      <div class="code-card-bar">
        <span class="mono">claude_desktop_config.json · .cursor/mcp.json</span>
        <button id="copy-config" type="button">Copiar</button>
      </div>
      <pre class="mono" id="mcp-config">
{
  "mcpServers": {
    "oracle-db": {
      "command": "/path/to/jdk/bin/java",
      "args": ["-jar", "/absolute/path/mcp-oracle-db-0.0.1-SNAPSHOT.jar"],
      "env": {
        "ORACLE_DB_URL": "jdbc:oracle:thin:@//db.host:1521/ORCLPDB1",
        "ORACLE_DB_USERNAME": "mcp_user",
        "ORACLE_DB_PASSWORD": "s3cret"
      }
    }
  }
}</pre
      >
    </div>

    <div class="final-cta">
      <a
        class="btn-primary"
        href="https://github.com/zademy/mcp-oracle-database"
        rel="noopener"
        >Ver el código fuente ↗</a
      >
    </div>
  </div>
</section>
```

- [ ] **Step 2: Añadir los estilos**

```css
.act-close {
  padding: 6rem 0;
  border-top: 1px solid var(--border);
  text-align: center;
}

.act-close .lede {
  margin-left: auto;
  margin-right: auto;
}

.code-card {
  margin: 2.5rem auto 0;
  max-width: 720px;
  border: 1px solid var(--border);
  border-radius: 1rem;
  background: #0b0f14;
  color: #e6edf3;
  overflow: hidden;
  text-align: left;
  box-shadow: 0 20px 50px -20px rgba(2, 6, 23, 0.5);
}

.code-card-bar {
  display: flex;
  align-items: center;
  justify-content: space-between;
  gap: 1rem;
  padding: 0.7rem 1rem;
  border-bottom: 1px solid rgba(255, 255, 255, 0.08);
  background: rgba(255, 255, 255, 0.03);
  font-size: 0.72rem;
  color: #8b949e;
}

.code-card-bar button {
  border: 1px solid rgba(255, 255, 255, 0.2);
  background: transparent;
  color: #e6edf3;
  border-radius: 0.5rem;
  padding: 0.3rem 0.8rem;
  font-size: 0.72rem;
  font-weight: 700;
  cursor: pointer;
}

.code-card pre {
  margin: 0;
  padding: 1.2rem 1.4rem;
  font-size: 0.8rem;
  line-height: 1.7;
  overflow-x: auto;
}

.final-cta {
  margin-top: 3rem;
}
```

- [ ] **Step 3: Añadir el botón copiar (funciona con o sin GSAP)**

Dentro de la IIFE principal, ANTES del `if (!window.gsap...)` return (debe
funcionar aunque GSAP no cargue):

```js
const copyBtn = document.getElementById("copy-config");
if (copyBtn) {
  copyBtn.addEventListener("click", async () => {
    const text = document.getElementById("mcp-config").textContent;
    try {
      await navigator.clipboard.writeText(text);
      copyBtn.textContent = "¡Copiado!";
    } catch {
      copyBtn.textContent = "Selecciona y copia manual";
    }
    setTimeout(() => (copyBtn.textContent = "Copiar"), 2000);
  });
}
```

Y la entrada del acto dentro de `mm.add(...)`:

```js
gsap.from(".code-card", {
  autoAlpha: 0,
  y: 40,
  scale: 0.97,
  duration: 0.8,
  ease: "power2.out",
  scrollTrigger: { trigger: ".act-close", start: "top 75%" },
});
```

- [ ] **Step 4: Verificar build + copiar**

Run: `npm run build` (éxito). En el dev server: botón "Copiar" pone el JSON en
el portapapeles y muestra "¡Copiado!".

- [ ] **Step 5: Commit**

```bash
git add src/pages/index.astro
git commit -m "feat(index): acto 6 con quickstart copiable y CTA final"
```

---

### Task 7: Guardas finales (reduced-motion, CDN caído) + verificación completa

**Files:**

- Modify: `src/pages/index.astro` (solo la IIFE GSAP)

**Interfaces:**

- Consume: todas las animaciones registradas en `mm.add(...)` de Tasks 1-6.

- [ ] **Step 1: Revisar la estructura de la IIFE**

La IIFE ya debe tener esta forma (confirmar y ajustar si falta):

```js
(function () {
  // 1) Botón copiar — independiente de GSAP (Task 6)

  if (!window.gsap || !window.ScrollTrigger) return; // CDN caído: página legible
  gsap.registerPlugin(ScrollTrigger);

  const mm = gsap.matchMedia();
  mm.add("(prefers-reduced-motion: no-preference)", () => {
    // Todos los timelines de los actos 1-6
    return () => {}; // cleanup de matchMedia
  });
})();
```

Verificar que NINGÚN `gsap.set(..., { autoAlpha: 0 })` esté fuera del bloque
`mm.add` — si alguno quedó fuera, moverlo dentro.

- [ ] **Step 2: Probar reduced-motion**

En DevTools del navegador: Rendering → Emulate CSS `prefers-reduced-motion: reduce`
→ recargar. Expected: todo el contenido visible de inmediato, sin animaciones,
sin secciones pinned que atrapen el scroll.

- [ ] **Step 3: Probar CDN caído**

En DevTools → Network bloquear `cdn.jsdelivr.net` → recargar.
Expected: página completa y legible, sin errores fatales en consola.

- [ ] **Step 4: Verificación completa**

```bash
npm run build
git status
```

Expected: build exitoso; `git status` muestra cambios SOLO en
`src/pages/index.astro` (+ docs del plan); `astro.config.mjs` y
`public/informes/` intactos.

- [ ] **Step 5: Probar en viewport móvil**

DevTools responsive a 390px: hero apilado, deck de herramientas en 1 columna,
pinned sections con scroll fluido.

- [ ] **Step 6: Commit final (si hubo ajustes)**

```bash
git add src/pages/index.astro
git commit -m "fix(index): guardas reduced-motion y CDN para scrollytelling"
```
