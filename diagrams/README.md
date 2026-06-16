# Diagramas

Todos os diagramas são **Mermaid** (renderizam nativo no GitHub dentro dos `../docs/`). Fontes em
[`src/`](src/) e imagens renderizadas em [`img/`](img/) (SVG + PNG, usadas na apresentação).

| Diagrama | Fonte | Uso |
|---|---|---|
| C4 — Contexto | `src/c4-context.mmd` | docs + slide |
| C4 — Container | `src/c4-container.mmd` | docs + slide |
| Infraestrutura (deployment AWS) | `src/infra-deployment.mmd` | docs + slide |
| Modelo de dados (ER) | `src/er-model.mmd` | docs + slide |
| Sequência — criação de demanda | `src/sequence-demanda.mmd` | docs |
| Intake inteligente | `src/intake.mmd` | docs + slide |
| Esteira CI/CD (detalhada) | `src/pipeline-cicd.mmd` | docs |
| Esteira CI/CD (slide, horizontal) | `src/pipeline-slide.mmd` | slide |
| GitFlow | `src/gitflow.mmd` | docs + slide |
| Escalabilidade | `src/escala.mmd` | docs + slide |
| Roadmap | `src/roadmap.mmd` | docs + slide |
| Segurança em camadas (detalhada) | `src/seguranca-camadas.mmd` | docs |
| Segurança em camadas (slide, horizontal) | `src/seguranca-slide.mmd` | slide |

> Padrão único: **Mermaid**. Versões `-slide` são variações horizontais otimizadas para caber em 16:9.

## Como (re)renderizar

Imagens geradas via [Kroki](https://kroki.io):

```bash
for f in src/*.mmd; do
  n=$(basename "$f" .mmd)
  curl -s -X POST https://kroki.io/mermaid/svg --data-binary @"$f" -o "img/$n.svg"
  curl -s -X POST https://kroki.io/mermaid/png --data-binary @"$f" -o "img/$n.png"
done
```
