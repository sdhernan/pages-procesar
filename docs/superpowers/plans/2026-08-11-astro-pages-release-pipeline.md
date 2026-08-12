# Astro GitHub Pages Deployment Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Publicar el sitio Astro en `https://sdhernan.github.io/pages-procesar/` desde `main` con GitHub Actions.

**Architecture:** `astro.config.mjs` define la URL canónica y el prefijo del project site. Un único workflow usa la acción oficial de Astro para instalar, compilar y subir el artifact; un segundo job lo despliega con la acción oficial de GitHub Pages.

**Tech Stack:** Astro 7.2.1, Node.js 22, npm, GitHub Actions, GitHub Pages.

## Global Constraints

- Desplegar únicamente desde `main` o mediante `workflow_dispatch`.
- Publicar en `https://sdhernan.github.io/pages-procesar/` sin dominio propio.
- No ejecutar semantic-release ni commitlint.
- No comitear `dist`, `artifact.tar` ni `.nojekyll`.
- Usar `withastro/action@v6`, `actions/checkout@v7` y `actions/deploy-pages@v5`.
- Mantener Node.js 22 porque `package.json` exige `node >=22.12.0`.

---

### Task 1: Configurar el project site y su despliegue

**Files:**
- Modify: `astro.config.mjs`
- Create: `.github/workflows/deploy.yml`
- Modify: `README.md`
- Modify: `package.json`
- Modify: `package-lock.json`
- Delete: `commitlint.config.cjs`

**Interfaces:**
- Consumes: script `build` de `package.json` y `package-lock.json`.
- Produces: artifact estático de Pages generado desde `dist/` y deployment en el environment `github-pages`.

- [x] **Step 1: Comprobar que el build actual no contiene el prefijo del repositorio**

Run: `npm run build`

Run:

```powershell
Select-String -Path dist\index.html -Pattern '/pages-procesar/'
```

Expected: el build termina correctamente, pero `Select-String` no encuentra el prefijo porque `astro.config.mjs` aún no define `base`.

- [x] **Step 2: Configurar Astro para la URL de GitHub Pages**

Reemplazar `astro.config.mjs` por:

```js
// @ts-check
import { defineConfig } from "astro/config";

// https://astro.build/config
export default defineConfig({
  site: "https://sdhernan.github.io",
  base: "/pages-procesar/",
});
```

- [x] **Step 3: Crear el workflow de deployment**

Crear `.github/workflows/deploy.yml` con:

```yaml
name: Deploy to GitHub Pages

on:
  push:
    branches: [main]
  workflow_dispatch:

permissions:
  contents: read

concurrency:
  group: pages
  cancel-in-progress: false

jobs:
  build:
    if: github.ref == 'refs/heads/main'
    runs-on: ubuntu-latest
    steps:
      - name: Checkout repository
        uses: actions/checkout@v7
      - name: Build and upload site
        uses: withastro/action@v6
        with:
          node-version: 22

  deploy:
    needs: build
    runs-on: ubuntu-latest
    permissions:
      pages: write
      id-token: write
    environment:
      name: github-pages
      url: ${{ steps.deployment.outputs.page_url }}
    steps:
      - name: Deploy to GitHub Pages
        id: deployment
        uses: actions/deploy-pages@v5
```

- [x] **Step 4: Alinear el README con el pipeline real**

Reemplazar la sección `## Integración continua` por:

```markdown
## Despliegue

El workflow `.github/workflows/deploy.yml` compila y publica el sitio en
GitHub Pages con cada push a `main`. También puede ejecutarse manualmente desde
la pestaña Actions.

La URL pública es <https://sdhernan.github.io/pages-procesar/>. En la
configuración del repositorio, `Settings > Pages > Build and deployment >
Source` debe estar en `GitHub Actions`.
```

- [x] **Step 5: Eliminar tooling fuera de alcance**

Eliminar `commitlint.config.cjs`, la sección `release` y las `devDependencies`
de commitlint/semantic-release de `package.json`.

Run: `npm install --package-lock-only`

Expected: `package-lock.json` conserva Astro como única dependencia raíz y ya
no resuelve commitlint ni semantic-release.

- [x] **Step 6: Validar la sintaxis del workflow**

Run:

```powershell
npx --yes yaml-lint .github/workflows/deploy.yml
```

Expected: salida sin errores de sintaxis YAML. Si `yaml-lint` informa solo diferencias de estilo no funcionales, corregirlas antes de continuar.

- [x] **Step 7: Compilar y comprobar las rutas generadas**

Run: `npm run build`

Expected: Astro termina con código 0 y crea `dist/index.html`.

Run:

```powershell
Select-String -Path dist\index.html -Pattern '/pages-procesar/favicon.svg','/pages-procesar/informes/'
```

Expected: aparecen referencias al favicon y a los tres informes con el prefijo `/pages-procesar/`.

- [x] **Step 8: Revisar el cambio y confirmar que no incluye artifacts**

Run: `git status --short`

Expected: solo aparecen `astro.config.mjs`, `.github/workflows/deploy.yml`,
`README.md`, `package.json`, `package-lock.json`, la eliminación de
`commitlint.config.cjs` y los documentos de diseño/plan ya acordados; `dist/`,
`artifact.tar` y `.nojekyll` no aparecen.

Run: `git diff --check`

Expected: no hay errores de whitespace.

- [x] **Step 9: Revisar y entregar sin hacer commit automático**

Run: `git diff -- astro.config.mjs README.md package.json package-lock.json commitlint.config.cjs`

Run: `git diff --no-index -- NUL .github/workflows/deploy.yml`

Expected: el diff contiene solo la configuración `site`/`base`, la eliminación
del tooling fuera de alcance, el workflow de dos jobs y la documentación
actualizada. El usuario hará commit y push cuando decida integrar el cambio en
`main`.

---

### Task 2: Verificación remota después de integrar en `main`

**Files:**
- No repository file changes.

**Interfaces:**
- Consumes: workflow `Deploy to GitHub Pages` integrado en `main` y Pages configurado con source `GitHub Actions`.
- Produces: sitio accesible públicamente en la URL acordada.

- [ ] **Step 1: Configurar la fuente de Pages**

En GitHub, abrir `Settings > Pages > Build and deployment` y seleccionar `GitHub Actions` como `Source`.

Expected: GitHub conserva la selección y muestra que Pages se desplegará mediante workflows.

- [ ] **Step 2: Observar el primer deployment**

Después del push a `main`, abrir el workflow `Deploy to GitHub Pages` en Actions.

Expected: los jobs `build` y `deploy` terminan correctamente y el deployment muestra `https://sdhernan.github.io/pages-procesar/`.

- [ ] **Step 3: Verificar el sitio público**

Abrir `https://sdhernan.github.io/pages-procesar/`.

Expected: carga la portada, aparece el favicon y los tres enlaces de informes abren sus páginas bajo `/pages-procesar/informes/`.
