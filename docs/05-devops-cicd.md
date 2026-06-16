# DevOps e Esteira CI/CD

## Visão geral

```mermaid
flowchart TD
    DEV[Dev: git push] --> PR[Pull Request]
    PR --> L
    subgraph CIB [Continuous Integration]
        L[Lint + Type check] --> T[Testes unit + integração]
        T --> COV[Cobertura 80%+]
        COV --> SAST[SAST + Quality Gate - SonarQube]
        SAST --> SCA[Scan dependências - Snyk]
        SCA --> SEC[Scan secrets - Gitleaks]
        SEC --> BUILD[Build imagem Docker multi-stage]
        BUILD --> IMG[Scan imagem - Trivy]
    end
    IMG -->|verde| MERGE[Merge develop]
    MERGE --> CDH[Deploy Homologação]
    CDH --> E2E[Testes E2E - Playwright]
    E2E --> SMOKE[Smoke tests + health check]
    SMOKE --> APPROVE{Aprovação manual + homologação}
    APPROVE -->|aprova| CDP[Deploy Produção - blue-green]
    CDP --> POST[Health check + métricas]
    POST -->|falha| RB[Rollback automático]
```

## Estágios (commit → produção)

| Estágio | Ferramentas | O que faz | Bloqueia |
|---|---|---|---|
| Commit | Husky + commitlint + lint-staged | Valida commit e lint local | ✅ |
| Lint/Type | ESLint, Prettier, `tsc` | Estilo e tipos | ✅ |
| Testes | Jest (unit + integração) | Lógica e contratos | ✅ |
| Cobertura | Jest --coverage | ≥ 80% no core | ✅ |
| SAST | SonarQube | Bugs, smells, vulns no código | ✅ |
| SCA | Snyk / `npm audit` | Vulns em dependências | ✅ |
| Secrets | Gitleaks | Token/senha vazado | ✅ |
| Build | Docker multi-stage | Imagem enxuta e reproduzível | ✅ |
| Scan imagem | Trivy | CVE na imagem base | ✅ |
| Deploy homolog | ArgoCD / Actions | Ambiente espelho de produção | auto |
| E2E | Playwright | Fluxos críticos | ✅ |
| Aprovação | manual + aceite | Gate humano (RN04) | ✅ |
| Deploy prod | blue-green / rolling | Sem downtime | — |
| Pós-deploy | health, Prometheus, Sentry | Observa e dispara rollback | auto |

## Estratégia de deploy

- **Blue-Green / Rolling** → zero downtime.
- **Migrations** versionadas e reversíveis (expand/contract para mudanças incompatíveis).
- **Feature flags** → desacoplam deploy de release.
- **Rollback automático** se o health check pós-deploy falhar.

## Ambientes

| Ambiente | Propósito | Deploy |
|---|---|---|
| dev | desenvolvimento local | Docker Compose |
| homologação | espelho de produção, E2E + aceite | automático no merge `develop` |
| produção | usuários reais | manual aprovado a partir de `main` |

## Escolha de ferramentas (CI/CD e IaC)

**Por que GitHub Actions e não a esteira nativa da AWS (CodePipeline)?** O código está no GitHub, então
o Actions integra nativo — roda os gates no próprio Pull Request, com review e status check onde o time
já trabalha, e faz deploy na AWS via **OIDC** (sem chave estática). A própria AWS **fechou o CodeCommit
para novos clientes em 2024**, sinalizando o fluxo "git no GitHub, deploy na AWS". O CodePipeline seria
defensável apenas se a empresa exigisse tudo dentro da AWS por isolamento/compliance.

**IaC com Terraform.** A infraestrutura (rede, RDS Postgres, ElastiCache Redis, ECS/Fargate, S3) é
declarada em código versionado com **Terraform** — padrão de mercado e **multi-cloud** (coerente com
evitar lock-in). O pipeline roda `terraform plan` (revisado no PR) e `apply`. **Pulumi** é uma alternativa
viável (IaC em TypeScript, mesma linguagem do projeto) — fica como menção honrosa, não como padrão, para
não fragmentar a stack. Cuidado operacional: **state remoto** (S3) com **lock** (DynamoDB) contra drift e
concorrência.

## IaC e observabilidade

- **Docker** multi-stage; **Terraform** provisiona a cloud; **secrets** em cofre (12-Factor).
- **Logs** (Pino → Loki), **métricas** (Prometheus + Grafana), **tracing** (OpenTelemetry),
  **erros** (Sentry), **alertas** (Grafana → Slack/WhatsApp).
