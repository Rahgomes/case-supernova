# Diagramas

Fontes versionadas em [`src/`](src/) e imagens renderizadas em [`img/`](img/).

| Diagrama | Tipo | Fonte | Imagem |
|---|---|---|---|
| C4 — Contexto | PlantUML (C4) | `src/c4-context.puml` | `img/c4-context.svg` |
| C4 — Container | Mermaid (C4) | `src/c4-container.mmd` · `src/c4-container.puml` | `img/c4-container.svg` |
| Modelo de dados (ER) | Mermaid | `src/er-model.mmd` | `img/er-model.svg` |
| Sequência — criação de demanda | Mermaid | `src/sequence-demanda.mmd` | `img/sequence-demanda.svg` |
| Intake inteligente | Mermaid | `src/intake.mmd` | `img/intake.svg` |
| Esteira CI/CD | Mermaid | `src/pipeline-cicd.mmd` | `img/pipeline-cicd.svg` |
| GitFlow | Mermaid | `src/gitflow.mmd` | `img/gitflow.svg` |
| Escalabilidade | Mermaid | `src/escala.mmd` | `img/escala.svg` |
| Roadmap | Mermaid | `src/roadmap.mmd` | `img/roadmap.svg` |
| Segurança em camadas | Mermaid | `src/seguranca-camadas.mmd` | `img/seguranca-camadas.svg` |

> Os diagramas Mermaid também renderizam **nativamente no GitHub** dentro dos documentos em [`../docs/`](../docs/).
> As imagens em `img/` (SVG e PNG) são usadas na apresentação.

## Como (re)renderizar

As imagens foram geradas via [Kroki](https://kroki.io). Exemplo:

```bash
curl -s -X POST https://kroki.io/mermaid/svg --data-binary @src/intake.mmd -o img/intake.svg
curl -s -X POST https://kroki.io/plantuml/svg --data-binary @src/c4-context.puml -o img/c4-context.svg
```

Os arquivos `.puml` (C4) também renderizam localmente com PlantUML: `java -jar plantuml.jar src/c4-context.puml`.
