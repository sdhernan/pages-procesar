# pages-procesar

Sitio Astro (GitHub Pages) con los resultados del proyecto de innovación
**MCP Server para Oracle**: tres informes comparativos 1 a 1 de la
optimización de la Query 01 (Desempleo IMSS, Desempleo ISSSTE y Matrimonio).

## Desarrollo

```bash
npm install
npm run dev      # http://localhost:4321
npm run build    # genera dist/
npm run preview  # sirve el sitio construido
```

## Despliegue

El workflow `.github/workflows/deploy.yml` compila y publica el sitio en
GitHub Pages con cada push a `main`. También puede ejecutarse manualmente desde
la pestaña Actions.

La URL pública es <https://sdhernan.github.io/pages-procesar/>. En la
configuración del repositorio, `Settings > Pages > Build and deployment >
Source` debe estar en `GitHub Actions`.
