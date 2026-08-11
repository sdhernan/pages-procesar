# Diseño: pipeline completo de release + deploy para pages-procesar

Fecha: 2026-08-11
Estado: aprobado
Alcance: `C:\Proyectos\pages-procesar` (Astro 7.2.1, npm, GitHub Pages)

## Objetivo

Completar el pipeline de CI/CD del sitio Astro:

1. Gate de calidad en Pull Requests: Conventional Commits + build.
2. Deploy a GitHub Pages en cada push a `main` (ya existe, se conserva).
3. Releases automáticos con versionado semántico a partir de los commits
   (semantic-release), con tag, GitHub Release y CHANGELOG.md.

## Decisiones

| Tema | Decisión | Motivo |
|------|----------|--------|
| Herramienta de release | semantic-release | Estándar empresarial; config declarativa en `package.json`; 100% automático en push |
| Estructura | Dos workflows (`ci.yml` + `deploy.yml`) | Separación de responsabilidades; PRs sin permisos de escritura |
| Orden en `main` | build → release → deploy | Solo se versiona y despliega código que compila (fail-fast) |
| npm | `"private": true`, sin publish | El proyecto es un sitio, no una librería |
| Commits | Conventional Commits validados en CI | Garantiza datos para el versionado |

Descartado: release-please (requiere aprobar PRs de release), release manual
(no cumple el requisito de automatización).

## Componentes

### 1. `.github/workflows/ci.yml` (nuevo)

Trigger: `pull_request` hacia `main`. Permisos: `contents: read`.
Concurrency: `ci-${{ github.ref }}` con `cancel-in-progress: true`.

- Job `commitlint`
  - `actions/checkout@v4` con `fetch-depth: 0`.
  - `actions/setup-node@v4`, Node 22, caché npm.
  - `npm ci`.
  - `npx commitlint --from ${{ github.event.pull_request.base.sha }} --to ${{ github.event.pull_request.head.sha }} --verbose`.
- Job `build`
  - Checkout, setup Node 22 con caché npm.
  - `npm ci` + `npm run build` (equivalente a `astro build`).
  - No requiere artifact; su fin es bloquear el merge si el sitio no compila.

### 2. `.github/workflows/deploy.yml` (existente, reordenado)

Trigger: `push` a `main` + `workflow_dispatch`. Permisos: `contents: write`,
`pages: write`, `id-token: write`. Concurrency de Pages sin cambios
(`group: pages`, `cancel-in-progress: false`).

- Job `build` (sin cambios funcionales respecto al actual):
  - Detección de package manager, setup Node 22, `configure-pages`,
    `npm ci`, `astro build` con `--site`/`--base`, upload del artifact `dist`.
- Job `release` (`needs: build`)
  - `actions/checkout@v4` con `fetch-depth: 0` (historia completa para analizar
    commits).
  - Setup Node 22 con caché npm, `npm ci`.
  - `npx semantic-release` con `env: GH_TOKEN: ${{ secrets.GITHUB_TOKEN }}`.
  - Efecto: crea tag semver, GitHub Release con notas, escribe CHANGELOG.md y
    comitea el bump de `version` en `package.json` de vuelta a `main`.
- Job `deploy` (`needs: [build, release]`): sin cambios
  (`actions/deploy-pages@v5`).

Nota de convergencia: el commit del bump dispara un segundo run del workflow;
en él no hay commits relevantes, semantic-release termina con éxito sin
publicar nada y el redeploy es idempotente.

### 3. `package.json` (modificación)

- Añadir `"private": true` (semantic-release hace bump local sin publicar a npm).
- devDependencies:
  - `semantic-release` (^24)
  - `@semantic-release/changelog` (^6)
  - `@semantic-release/git` (^10)
  - `@commitlint/cli` (^19)
  - `@commitlint/config-conventional` (^19)
- Llave `"release"`:

```json
{
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

Nota: `commit-analyzer`, `release-notes-generator` y `github` vienen incluidos
en semantic-release; se declaran explícitos para fijar el orden de ejecución.

### 4. `commitlint.config.cjs` (nuevo)

El proyecto usa `"type": "module"`, así que la config CommonJS va en `.cjs`:

```js
module.exports = { extends: ['@commitlint/config-conventional'] };
```

## Comportamiento esperado

| Prefijo de commit | Efecto |
|-------------------|--------|
| `feat:` | minor (x.Y+1.0) + release |
| `fix:` | patch (x.y.Z+1) + release |
| `feat!:` o `BREAKING CHANGE` | major (X+1.0.0) + release |
| `chore:`, `docs:`, `style:`, `ci:` | sin release; deploy sí ocurre |

- Primer release: será `v1.0.0` con el primer commit `feat:` convencional
  (los commits históricos no siguen el formato y no generan release).
- Si el job `build` falla, no hay release ni deploy.
- Si no hay commits relevantes, semantic-release sale con éxito y el deploy
  continúa.

## Plan de verificación

1. `npm install` local tras los cambios (verifica resolución de versiones).
2. `npx commitlint --from HEAD~1` con un mensaje conforme y uno inválido.
3. `npx semantic-release --dry-run` en `main` (no debe publicar).
4. Tras el push: observar el primer run de Actions (build → release → deploy)
   y confirmar que el segundo run converge sin nuevo release.
5. Crear un commit `fix: prueba pipeline` y confirmar tag + Release en GitHub.

## Riesgos

- El segundo run por el commit de bump es ruido esperado, no un fallo.
- Commits no convencionales en PRs serán rechazados por `commitlint`; el
  equipo debe adoptar el formato (mitigación: mensaje claro del job).
- Los pushes directos a `main` evaden el gate de `commitlint` (corre solo en
  PRs). Mitigación opcional: regla de protección de rama que exija PR para
  integrar a `main`.
