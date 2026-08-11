# pages-procesar

Sitio Astro (GitHub Pages) con los resultados del proyecto de innovación
**MCP Server para Oracle**: tres informes comparativos 1 a 1 de la
optimización de la Query 01 (Desempleo IMSS, Desempleo ISSSTE y Matrimonio).

## Desarrollo

```bash
npm install
npm run dev      # http://localhost:4321
npm run build    # genera dist/
```

## Despliegue

El workflow `.github/workflows/deploy.yml` construye y publica en GitHub
Pages en cada push a `main`. Los informes estáticos viven en
`public/informes/`.
