# Astro Pages Release Pipeline — Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Completar el pipeline CI/CD de `pages-procesar`: gate de PR (commitlint + build), releases automáticos con semantic-release y deploy a GitHub Pages.

**Architecture:** Dos workflows: `ci.yml` (pull_request, read-only) y `deploy.yml` extendido con orden build → release → deploy en `main`. La configuración de semantic-release vive en `package.json`; commitlint usa config CommonJS `.cjs`.

**Tech Stack:** GitHub Actions, semantic-release ^24, @semantic-release/changelog ^6, @semantic-release/git ^10, @commitlint/cli ^19, Astro 7.2.1, Node 22, npm.

## Global Constraints

- Proyecto raíz: `C:\Proyectos\pages-procesar` (rama `main`, npm con `package-lock.json`).
- Shell local: Windows cmd (nada de sintaxis bash en comandos locales).
- Node local >= 22.12.0 (exigido por `engines` del proyecto y por Astro 7).
- El sitio NO se publica a npm: `package.json` lleva `"private": true` y el plugin npm con `npmPublish: false`.
- Solo se versiona desde `main` (`"branches": ["main"]`).
- Los commits hechos durante este plan usan formato Conventional Commits (`ci:`, `feat:`, etc.).
- No commitear `node_modules` ni `dist` (el `.gitignore` ya los cubre).

---

### Task 1: package.json + commitlint config + dependencias

**Files:**
- Modify: `package.json`
- Create: `commitlint.config.cjs`

**Interfaces:**
- Produce: llave `"release"` en `package.json` (consumida por el job `release` de Task 3), config commitlint (consumida por el job `commitlint` de Task 2).

- [ ] **Step 1: Instalar devDependencies**

Run (desde `C:\Proyectos\pages-procesar`):

```cmd
npm install -D semantic-release@^24 @semantic-release/changelog@^6 @semantic-release/git@^10 @commitlint/cli@^19 @commitlint/config-conventional@^19
```

Expected: instala sin errores; `package.json` gana la sección `devDependencies` con las 5 librerías y `package-lock.json` se actualiza.

- [ ] **Step 2: Añadir `private` y la llave `release` a `package.json`**

Resultado final esperado del archivo (la versión de astro y scripts no cambian; las versiones exactas de devDependencies serán las resueltas en Step 1 — NO reescribirlas a mano):

```json
{
  "name": "pages-procesar",
  "private": true,
  "type": "module",
  "version": "0.0.1",
  "engines": {
    "node": ">=22.12.0"
  },
  "scripts": {
    "dev": "astro dev",
    "build": "astro build",
    "preview": "astro preview",
    "astro": "astro"
  },
  "dependencies": {
    "astro": "^7.2.1"
  },
  "devDependencies": {
    ...resuelto por npm en Step 1...
  },
  "release": {
    "branches": ["main"],
    "plugins": [
      "@semantic-release/commit-analyzer",
      "@semantic-release/release-notes-generator",
      "@semantic-release/changelog",
      ["@semantic-release/npm", { "npmPublish": false }],
      "@semantic-release/github",
      "@semantic-release/git"
    ]
  }
}
```

Cambios concretos sobre el archivo existente:
1. Añadir `"private": true` después de `"name"`.
2. Añadir la llave `"release"` al final del objeto raíz (después de `devDependencies`).

- [ ] **Step 3: Crear `commitlint.config.cjs`**

El proyecto usa `"type": "module"`; la config CommonJS DEBE ir en `.cjs`:

```js
module.exports = { extends: ['@commitlint/config-conventional'] };
```

- [ ] **Step 4: Verificar commitlint — mensaje válido pasa**

Run:

```cmd
echo feat: prueba de mensaje valido | npx commitlint --verbose
```

Expected: salida `✔ subject must not be empty`, `✔ type must be one of [...]` y exit code 0.

- [ ] **Step 5: Verificar commitlint — mensaje inválido falla**

Run:

```cmd
echo mensaje sin formato | npx commitlint
```

Expected: exit code 1 con error `type may not be null`.

- [ ] **Step 6: Commit**

```cmd
git add package.json package-lock.json commitlint.config.cjs docs\superpowers
git commit -m "ci: add semantic-release and commitlint configuration"
```

(Incluye `docs\superpowers` para commitear el spec y este plan.)

---

### Task 2: workflow `ci.yml` (gate de PR)

**Files:**
- Create: `.github/workflows/ci.yml`

**Interfaces:**
- Consume: `commitlint.config.cjs` y devDependencies de Task 1.

- [ ] **Step 1: Crear `.github/workflows/ci.yml`**

```yaml
name: CI

on:
  pull_request:
    branches: ["main"]

permissions:
  contents: read

concurrency:
  group: ci-${{ github.ref }}
  cancel-in-progress: true

jobs:
  commitlint:
    name: Commit messages
    runs-on: ubuntu-latest
    steps:
      - name: Checkout
        uses: actions/checkout@v4
        with:
          fetch-depth: 0
      - name: Setup Node
        uses: actions/setup-node@v4
        with:
          node-version: "22"
          cache: npm
      - name: Install dependencies
        run: npm ci
      - name: Validate commit messages
        run: npx commitlint --from "${{ github.event.pull_request.base.sha }}" --to "${{ github.event.pull_request.head.sha }}" --verbose

  build:
    name: Build
    runs-on: ubuntu-latest
    steps:
      - name: Checkout
        uses: actions/checkout@v4
      - name: Setup Node
        uses: actions/setup-node@v4
        with:
          node-version: "22"
          cache: npm
      - name: Install dependencies
        run: npm ci
      - name: Build with Astro
        run: npm run build
```

- [ ] **Step 2: Validar sintaxis YAML**

Run:

```cmd
npx --yes js-yaml .github\workflows\ci.yml > NUL
```

Expected: sin errores (js-yaml imprime JSON a NUL; un YAML inválido aborta con error de parseo).

- [ ] **Step 3: Commit**

```cmd
git add .github\workflows\ci.yml
git commit -m "ci: add pull request gate with commitlint and build"
```

---

### Task 3: reordenar `deploy.yml` (build → release → deploy)

**Files:**
- Modify: `.github/workflows/deploy.yml`

**Interfaces:**
- Consume: llave `"release"` de `package.json` (Task 1).
- Produce: job `release` que crea tag, GitHub Release, CHANGELOG.md y commit de bump en `main`.

- [ ] **Step 1: Modificar `deploy.yml`**

Aplicar exactamente estos 4 cambios al archivo existente:

1. Cambiar el `name` del workflow:

```yaml
name: Release and deploy Astro site to Pages
```

2. Ampliar permisos (`contents` pasa de `read` a `write` — semantic-release necesita crear tags/releases y commitear el bump):

```yaml
permissions:
  contents: write
  pages: write
  id-token: write
```

3. Añadir este job NUEVO entre `build` y `deploy`:

```yaml
  release:
    name: Release
    needs: build
    runs-on: ubuntu-latest
    steps:
      - name: Checkout
        uses: actions/checkout@v4
        with:
          fetch-depth: 0
      - name: Setup Node
        uses: actions/setup-node@v4
        with:
          node-version: "22"
          cache: npm
      - name: Install dependencies
        run: npm ci
      - name: Semantic release
        run: npx semantic-release
        env:
          GH_TOKEN: ${{ secrets.GITHUB_TOKEN }}
```

4. Cambiar la dependencia del job `deploy`:

```yaml
    needs: [build, release]
```

Todo lo demás del archivo (detección de package manager, `configure-pages`, build con `--site`/`--base`, upload del artifact, concurrency de Pages) queda IGUAL.

- [ ] **Step 2: Validar sintaxis YAML**

Run:

```cmd
npx --yes js-yaml .github\workflows\deploy.yml > NUL
```

Expected: sin errores.

- [ ] **Step 3: Commit**

```cmd
git add .github\workflows\deploy.yml
git commit -m "ci: add semantic-release job before pages deploy"
```

---

### Task 4: verificación local end-to-end

**Files:** ninguno (solo verificación).

**Interfaces:**
- Consume: resultados de Tasks 1-3.

- [ ] **Step 1: El sitio compila**

Run:

```cmd
npm run build
```

Expected: `astro build` completa sin errores y genera `dist\`.

- [ ] **Step 2: Dry-run de semantic-release**

Run:

```cmd
npx semantic-release --dry-run
```

Expected: analiza la historia de `main` SIN publicar ni commitear nada.
Caso probable: como los commits históricos no son convencionales, imprime
`There are no relevant changes, so the new release won't be published` — eso
es correcto: el primer release (`v1.0.0`) llegará con el primer commit
convencional tipo `feat:`/`fix:`.
Si en cambio se ve una tabla de análisis de commits y next version, también es
correcto. Cualquier stacktrace es un fallo a corregir.

- [ ] **Step 3: No hay commit nuevo** (el dry-run no debe tocar nada)

Run:

```cmd
git status
```

Expected: árbol limpio salvo cambios ya commiteados en Tasks 1-3.

---

### Task 5: push y verificación en GitHub (pasos manuales del usuario)

**Files:** ninguno.

- [ ] **Step 1: Push de los commits**

```cmd
git push origin main
```

Nota: el primer push usa commits con prefijo `ci:`, que NO generan release
(según diseño). El pipeline se estrena con un release real en el primer
commit `feat:`/`fix:`.

- [ ] **Step 2: Verificar el run de `deploy.yml`**

En GitHub → Actions: el run debe mostrar `Build ✔ → Release ✔ → Deploy ✔`.
En el log del job Release se espera `There are no relevant changes` (aún no
hay commits convencionales de usuario).

- [ ] **Step 3: Estrenar el release con un commit convencional**

Hacer un cambio trivial (p. ej. editar `README.md`) y:

```cmd
git add README.md
git commit -m "feat: document release pipeline"
git push origin main
```

- [ ] **Step 4: Confirmar el release**

Expected: en el segundo run, el job Release crea el tag `v1.0.0`, la entrada
en Releases con notas generadas, commitea `CHANGELOG.md` + bump de versión a
`main`, y el sitio queda desplegado en Pages. El tercer run (disparado por el
commit de bump) converge sin crear otro release — comportamiento esperado.

- [ ] **Step 5: Opcional — probar el gate de PR**

Crear una rama con un commit de mensaje inválido (`cambio random`), abrir un
PR a `main` y confirmar que el job `Commit messages` falla.
