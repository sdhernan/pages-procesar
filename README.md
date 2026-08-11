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

## Integración continua

El workflow `.github/workflows/ci.yml` se ejecuta en cada pull request a
`main`: valida los mensajes de commit con commitlint y compila el sitio con
Astro. Los informes estáticos viven en `public/informes/`.
