# Modelagem de Dados

Modelo relacional normalizado (3FN) com **histórico de negócio** e **trilha de auditoria** como
cidadãos de primeira classe — ambos append-only.

## Diagrama Entidade-Relacionamento

```mermaid
erDiagram
    DEPARTMENT ||--o{ USER : "tem"
    ROLE ||--o{ USER_ROLE : "concede"
    USER ||--o{ USER_ROLE : "possui"
    USER ||--o{ DEMAND : "abre (solicitante)"
    USER ||--o{ DEMAND : "executa (responsável)"
    DEPARTMENT ||--o{ DEMAND : "origem"
    DEMAND_CATEGORY ||--o{ DEMAND : "classifica"
    STATUS ||--o{ DEMAND : "estado atual"
    PRIORITY ||--o{ DEMAND : "prioridade"
    DEMAND ||--o{ COMMENT : "recebe"
    USER ||--o{ COMMENT : "escreve"
    DEMAND ||--o{ ATTACHMENT : "anexa"
    DEMAND ||--o{ DEMAND_HISTORY : "registra"
    DEMAND ||--o| SLA : "rege"
    SLA_POLICY ||--o{ SLA : "define"
    AUDIT_LOG }o--|| USER : "ação de"

    DEPARTMENT {
        uuid id PK
        string nome
        string codigo UK
        boolean ativo
    }
    ROLE {
        uuid id PK
        string nome UK
        jsonb permissoes
    }
    USER {
        uuid id PK
        string nome
        string email UK
        string senha_hash "argon2id"
        uuid department_id FK
        boolean ativo
        timestamp ultimo_acesso
    }
    USER_ROLE {
        uuid user_id FK
        uuid role_id FK
    }
    DEMAND_CATEGORY {
        uuid id PK
        string nome
        uuid sla_policy_id FK
    }
    PRIORITY {
        uuid id PK
        string nome "P0..P3"
        int peso
    }
    STATUS {
        uuid id PK
        string nome
        int ordem
        boolean is_final
    }
    DEMAND {
        uuid id PK
        string titulo
        text descricao
        uuid department_id FK
        uuid category_id FK
        uuid priority_id FK
        uuid status_id FK
        uuid solicitante_id FK
        uuid responsavel_id FK
        jsonb metadata
        timestamp criado_em
        timestamp atualizado_em
    }
    COMMENT {
        uuid id PK
        uuid demand_id FK
        uuid author_id FK
        text conteudo
        boolean interno
        timestamp criado_em
    }
    ATTACHMENT {
        uuid id PK
        uuid demand_id FK
        string nome_arquivo
        string storage_key
        string mime_type
        int tamanho_bytes
        uuid uploaded_by FK
    }
    DEMAND_HISTORY {
        uuid id PK
        uuid demand_id FK
        uuid author_id FK
        string campo
        string valor_anterior
        string valor_novo
        timestamp criado_em
    }
    SLA_POLICY {
        uuid id PK
        uuid priority_id FK
        int tempo_resposta_min
        int tempo_resolucao_min
        jsonb calendario
    }
    SLA {
        uuid id PK
        uuid demand_id FK
        uuid sla_policy_id FK
        timestamp inicio
        timestamp prazo_resposta
        timestamp prazo_resolucao
        string status_sla
    }
    AUDIT_LOG {
        uuid id PK
        uuid actor_id FK
        string acao
        string entidade
        uuid entidade_id
        jsonb contexto
        string ip
        timestamp criado_em
    }
```

## Decisões de modelagem

| Decisão | Porquê |
|---|---|
| **UUID** como PK | Evita enumeração (segurança); facilita sharding na escala |
| **STATUS/PRIORITY/CATEGORY** como tabelas | Workflow configurável sem deploy; relatórios consistentes |
| **DEMAND_HISTORY** append-only | Histórico imutável de mudanças de negócio |
| **AUDIT_LOG** separado | Auditoria de segurança/compliance (LGPD) distinta do histórico de negócio |
| **metadata jsonb** | Campos específicos por tipo sem alterar schema |
| **SLA vs SLA_POLICY** | Política é o template; SLA é a instância calculada por demanda |
| **COMMENT.interno** | Separa comentário interno de TI do visível ao solicitante |
| **Anexos em S3** | Não inflar o RDBMS; performance e custo |
| **USER_ROLE N:N** | Usuário pode ter múltiplos papéis |

## Índices e performance

```sql
CREATE INDEX idx_demand_status_priority ON demand (status_id, priority_id);
CREATE INDEX idx_demand_department ON demand (department_id);
CREATE INDEX idx_demand_responsavel ON demand (responsavel_id) WHERE responsavel_id IS NOT NULL;
CREATE INDEX idx_demand_fts ON demand USING gin (to_tsvector('portuguese', titulo || ' ' || descricao));
CREATE INDEX idx_history_demand ON demand_history (demand_id, criado_em DESC);
CREATE INDEX idx_audit_actor_date ON audit_log (actor_id, criado_em DESC);
CREATE INDEX idx_sla_status ON sla (status_sla, prazo_resolucao) WHERE status_sla <> 'violado';
```

## Integridade e estratégias

- **Soft delete** em `USER`/`DEPARTMENT` (`ativo=false`) — preserva histórico e atende LGPD.
- **Transação** ao mudar status: `UPDATE demand` + `INSERT demand_history` no mesmo commit.
- **Outbox pattern** para eventos (notificação, IA): evento gravado na mesma transação → worker publica.
- **Row-Level Security (RLS)** opcional para isolar dados por departamento.
