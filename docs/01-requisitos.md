# Requisitos

## Atores

| Ator | Descrição | Principais ações |
|---|---|---|
| **Solicitante** | Funcionário de área de negócio | Abre demanda, acompanha, comenta, anexa |
| **Triador / Gestor de TI** | Recebe e prioriza | Triagem, definição de SLA, atribuição |
| **Desenvolvedor / Responsável** | Executa a demanda | Atualiza status, registra esforço, homologa |
| **Aprovador / Líder de área** | Aprova demandas de alto impacto | Aprova/reprova, prioridade de negócio |
| **Admin** | Governança | Gerencia usuários, perfis, departamentos, SLA |
| **Sistema/Bot (IA)** | Intake automático | Recebe do WhatsApp, classifica, prioriza, cria rascunho |

## Requisitos Funcionais (MoSCoW)

`M`=Must · `S`=Should · `C`=Could

### Identidade e Acesso
- **RF01** `M` — Cadastro/gestão de usuários vinculados a **departamento** e **perfil**.
- **RF02** `M` — Autenticação corporativa; **SSO via OIDC** `S`.
- **RF03** `M` — Autorização baseada em papéis (**RBAC**).

### Ciclo de vida da Demanda
- **RF04** `M` — Abertura com título, descrição, departamento, categoria, prioridade sugerida, anexos.
- **RF05** `M` — **Workflow configurável**: `Aberta → Triada → Em Análise → Aprovada → Em Desenvolvimento → Homologação → Concluída / Rejeitada / Cancelada`.
- **RF06** `M` — **Backlog estruturado** com filtros (departamento, status, prioridade, responsável, SLA).
- **RF07** `M` — **Priorização formal** (matriz impacto × urgência → P0–P3).
- **RF08** `M` — Atribuição e reatribuição de responsável com registro.
- **RF09** `M` — **Comentários** por demanda (com flag interno).
- **RF10** `M` — **Anexos** (upload de arquivos).
- **RF11** `M` — **Histórico/timeline** de mudanças (quem, o quê, quando, antes→depois).
- **RF12** `S` — Notificações (e-mail, WhatsApp, in-app).

### SLA
- **RF13** `M` — Definição de **SLA por prioridade/tipo** (tempo de resposta e de resolução).
- **RF14** `M` — Cálculo de prazo, alerta de **SLA em risco** e marcação de **violação**.

### Intake Inteligente
- **RF15** `S` — Captura de demanda via **WhatsApp Business API**.
- **RF16** `C` — **Classificação e priorização automática por IA**.
- **RF17** `S` — Captura via e-mail (parser → rascunho).

### Visão e Gestão
- **RF18** `S` — Dashboard de métricas (status, SLA %, lead time, throughput).
- **RF19** `C` — Relatórios exportáveis (CSV/PDF).
- **RF20** `M` — **Trilha de auditoria** imutável.

## Requisitos Não Funcionais

| # | Categoria | Requisito | Como atender |
|---|---|---|---|
| **RNF01** | Performance | p95 da API < 300ms; backlog < 500ms | Índices, paginação, cache Redis |
| **RNF02** | Escalabilidade | 100 → 50.000 usuários sem reescrita | App stateless + scale horizontal + read replicas |
| **RNF03** | Disponibilidade | 99,9% uptime | Health checks, múltiplas instâncias, deploy sem downtime |
| **RNF04** | Segurança | TLS 1.3 em trânsito, AES-256 em repouso | Ver [segurança](06-seguranca.md) |
| **RNF05** | LGPD | Consentimento, minimização, exclusão, log de PII | Anonimização, retenção parametrizada |
| **RNF06** | Auditabilidade | 100% das ações sensíveis rastreáveis e imutáveis | Tabela append-only + outbox |
| **RNF07** | Observabilidade | Logs estruturados, métricas, tracing | OpenTelemetry, Prometheus/Grafana, Sentry |
| **RNF08** | Manutenibilidade | Cobertura ≥ 80% no core; lint obrigatório | Quality gates no CI |
| **RNF09** | Usabilidade | Responsivo, acessível (WCAG 2.1 AA) | Next.js + design system |
| **RNF10** | Portabilidade | Sem lock-in forte de cloud | Docker, 12-factor, S3-compatible |
| **RNF11** | Confiabilidade | Entregas idempotentes | Outbox pattern, filas com retry |

## Regras de negócio-chave

- **RN01** — Demanda P0 exige **aprovação de líder** antes do desenvolvimento.
- **RN02** — SLA começa a contar **na triagem**, não na abertura.
- **RN03** — Só Triador/Admin alteram prioridade; Solicitante apenas sugere.
- **RN04** — Demanda só fecha após **homologação do solicitante**.
- **RN05** — Anexos: limite 25MB, tipos permitidos (sem executáveis).
- **RN06** — Toda mudança de status gera **evento de histórico** imutável.

## Fora de escopo (inicial)

Gestão financeira de projetos, timesheet completo e integração com folha — previstos no roadmap.
