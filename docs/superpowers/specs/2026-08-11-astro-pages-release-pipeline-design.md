# Diseño: despliegue de Astro en GitHub Pages

Fecha: 2026-08-11
Estado: implementado; despliegue remoto pendiente
Alcance: `pages-procesar` (Astro 7.2.1, npm, GitHub Pages)

## Objetivo

Compilar y publicar el sitio Astro en
`https://sdhernan.github.io/pages-procesar/` mediante GitHub Actions.

El despliegue se ejecutará con cada push a `main` y también podrá iniciarse
manualmente desde la pestaña Actions. Este pipeline no generará versiones,
tags, GitHub Releases ni cambios en `CHANGELOG.md`.

## Decisiones

| Tema | Decisión | Motivo |
|------|----------|--------|
| Rama de despliegue | `main` | Solo se publica código integrado |
| URL | GitHub Pages estándar | No se necesita dominio propio ni `CNAME` |
| Implementación | Acción oficial de Astro | Reduce YAML y mantenimiento |
| Alcance | Solo build y deploy | Releases y commitlint quedan fuera del objetivo |
| Artifact | Artifact nativo de Pages | Evita comitear `dist` u otros archivos generados |

Se descartan dos alternativas:

- Pasos explícitos con `setup-node`, `npm ci`, `astro build` y
  `upload-pages-artifact`: ofrecen más control, pero duplican lo que ya hace la
  acción oficial de Astro.
- Publicación mediante una rama `gh-pages` o commits de `dist` en `main`:
  introduce escrituras y archivos generados innecesarios.

## Componentes

### `astro.config.mjs`

Astro se configurará como un project site de GitHub Pages:

```js
export default defineConfig({
  site: "https://sdhernan.github.io",
  base: "/pages-procesar/",
});
```

`import.meta.env.BASE_URL`, ya usado por `src/pages/index.astro`, incluirá el
prefijo del repositorio y el slash final que esperan los enlaces concatenados
a informes y al favicon.

### `.github/workflows/deploy.yml`

El workflow tendrá estos disparadores:

- `push` a `main`.
- `workflow_dispatch` para despliegues manuales desde `main`.

El job de build comprobará `github.ref` para impedir que una ejecución manual
seleccione y publique otra rama.

Usará permisos mínimos por job:

- `build`: `contents: read` para leer el repositorio.
- `deploy`: `pages: write` para crear el despliegue e `id-token: write` para
  autenticarlo; los demás permisos quedan en `none`.

La concurrencia usará el grupo `pages` y no cancelará un despliegue en curso.
El workflow tendrá dos jobs:

1. `build`: checkout del repositorio y ejecución de `withastro/action@v6`, que
   instala dependencias, ejecuta el script `build` y sube el artifact de Pages.
2. `deploy`: depende de `build`, usa el environment `github-pages` y publica el
   artifact con `actions/deploy-pages@v5`.

El checkout usará `actions/checkout@v7`, según el ejemplo oficial vigente de
Astro. El job de build fijará Node 22 porque `package.json` exige
`node >=22.12.0`.

## Flujo

1. Un cambio llega a `main` o se inicia el workflow manualmente.
2. GitHub Actions instala las dependencias con el lockfile de npm.
3. Astro genera `dist/` con el prefijo `/pages-procesar`.
4. La acción de Astro sube el artifact para Pages.
5. GitHub Pages despliega el artifact en el environment `github-pages`.
6. GitHub muestra la URL publicada en el job `deploy`.

Si la instalación o el build fallan, el job de despliegue no se ejecuta. Si un
nuevo push llega durante otro run, la concurrencia evita despliegues paralelos
sin interrumpir el que ya comenzó.

## Verificación

1. Validar la sintaxis del workflow.
2. Ejecutar `npm run build` localmente.
3. Confirmar que `dist/index.html` referencia
   `/pages-procesar/informes/` y `/pages-procesar/favicon.svg`.
4. Revisar el diff para comprobar que no se agregaron artifacts generados.
5. Después del push a `main`, verificar en GitHub Actions que `build` y
   `deploy` terminen correctamente.
6. Abrir `https://sdhernan.github.io/pages-procesar/` y comprobar la página,
   el favicon y los tres informes.

## Configuración externa

En GitHub, `Settings > Pages > Build and deployment > Source` debe estar en
`GitHub Actions`. Esta selección no puede declararse desde el workflow.

## Fuera de alcance

- semantic-release, commitlint, tags, releases y `CHANGELOG.md`; su
  configuración y dependencias se eliminan del proyecto.
- Validación de pull requests o Conventional Commits.
- Dominio personalizado.
- Comitear `dist`, `artifact.tar` o `.nojekyll` en `main`.
