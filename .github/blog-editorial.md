# Automatización editorial del blog

El workflow **Blog editorial con Codex** se ejecuta los lunes a las 09:15 en `America/Santiago`. También se puede ejecutar con **Actions → Blog editorial con Codex → Run workflow** en `main`.

## Activación

1. Guarda una [clave de un proyecto de OpenAI Platform](https://platform.openai.com/api-keys) con acceso a `gpt-6.2-sol` como secreto de repositorio `OPENAI_API_KEY` en **Settings → Secrets and variables → Actions**. La [acción oficial de Codex](https://learn.chatgpt.com/docs/github-action) usa esta clave y el consumo se registra en el proyecto de API.
2. En **Settings → Actions → General → Workflow permissions**, activa **Allow GitHub Actions to create and approve pull requests**. El workflow usa `GITHUB_TOKEN` para crear el issue y el pull request.
3. Una vez integrado el workflow en `main`, ejecútalo manualmente para crear el primer issue.

## Uso

Cada ejecución lee los títulos, descripciones y etiquetas de artículos de `src/content/blog` que pasarían el filtro de publicación de Astro. Codex usa ese catálogo y las propuestas recientes para generar 10 temas. Los títulos siguen el idioma del blog y cada propuesta incluye un enfoque en español.

Desde la cuenta propietaria del repositorio, responde al issue con el título completo de una propuesta. El workflow genera el artículo en inglés, formatea el Markdown y ejecuta el build de Astro. Si el build pasa, abre un pull request en borrador con un solo archivo nuevo en `src/content/blog`. Puedes revisar y publicar el artículo cuando decidas integrar el PR. Si GitHub solicita aprobar los checks del PR creado por el bot, puedes iniciarlos desde el propio PR.

Los comentarios repetidos para un mismo tema e issue reutilizan el PR existente. Para repetir una ejecución fallida, usa **Re-run failed jobs** en el historial de Actions.
