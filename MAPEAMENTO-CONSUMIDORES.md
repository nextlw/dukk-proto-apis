# 🗺️ MAPEAMENTO DE CONSUMIDORES — dukk-proto-apis

> Gerado em: 07/08/2026
> Objetivo: mapear quem **usa** o contrato protobuf (`dukk-proto-apis`), quem **não usa mas deveria**, e onde existem **definições duplicadas** de mensagens do contrato.
> Escopo varrido: `dukk` (desktop), `dukk-code` (backend/monorepo) e `SISTEMA-INTERNO` (CRM interno, Go).

---

## 1. Estado geral (o resumo da ópera)

| Repo | Linguagem | Papel | Usa proto? |
|---|---|---|---|
| `dukk` | Rust | Desktop (app do multi-agente) | 🟡 Parcial — bindings declarados no workspace, mas uso real só em testes + conversão de filtros |
| `dukk-code` | Rust + Python | Backend/monorepo (`dukk-core` server, `hermes-agent`, `agno`, SDKs) | 🔴 NÃO usa nada |
| `SISTEMA-INTERNO` | Go | CRM interno (Nokk + Integração), consome a API REST do dukk | 🔴 NÃO usa os bindings Go (`gen/go`) |

**Veredito:** o contrato existe, está versionado em 3 linguagens (Go, Rust, Python), mas o **único uso real em runtime é a conversão de filtros no desktop**. O restante do ecossistema define as mesmas mensagens à mão (serde/Rust, structs Go, JSON) — muitas vezes **em duplicata, em lugares diferentes**.

---

## 2. Por mensagem do contrato → quem deveria usar vs. quem duplica

### 2.1 `kanban.proto` (KanbanBoard, KanbanCard, KanbanRun, KanbanCardStatus…)

| Onde | Situação | Evidência |
|---|---|---|
| `dukk-code/dukk-core/crates/kanban/src/schema.rs` | 🔴 DUPLICA na mão (Rust serde) | `pub enum KanbanCardStatus { Triage, Todo, Ready, Running, Blocked, Done, Archived, Failed }`, `pub struct KanbanBoard`, `pub struct KanbanCard`, `pub struct KanbanRun` (linhas 13–52) |
| `dukk-code/dukk-core/crates/server/src/routes/agent_runs.rs` | 🔴 Mapeia status via `match` de string | `/// Mapeia o status do kanban_card para o estado do contrato Warp.` → `fn kanban_status_to_state(status: &str) -> &'static str` (linhas 169–170) |
| `dukk-code/dukk-core/crates/server/src/kanban_bridge.rs` | 🔴 Serializa tudo com `serde_json::json!` | linhas 291–319 |
| `dukk/app/src/kanban/types.rs` | 🔴 3ª definição duplicada (desktop) | `KanbanBoard`, `KanbanCard`, `KanbanComment`, `KanbanEvent`, `KanbanEventKind`, `CreateBoardRequest`, `EnqueueCardRequest`, `MoveCardRequest`… |
| `dukk/app/src/dukk/client.rs` | 🔴 Deserializa `serde_json::from_str::<KanbanBoard>` (linha 685) | contrato em JSON na mão |
| `dukk-proto-apis` `kanban.proto` | ✅ Fonte da verdade | já tem bindings Rust (`dukk_multi_agent_api`) |

> ⚠️ O Kanban existe em **3 definições distintas** (proto + backend + desktop) que precisam ser mantidas sincronizadas à mão. É o pior caso do ecossistema.

### 2.2 `orchestration.proto` (OrchestrationConfig, Harness, execution_mode Local/Remote)

| Onde | Situação | Evidência |
|---|---|---|
| `dukk/crates/ai/src/agent/orchestration_config.rs` | 🔴 DUPLICA (confessado) | `/// Mirrors the proto OrchestrationConfig but uses Rust-native types to keep view / model code free of proto imports.` — struct `OrchestrationConfig` + enum `OrchestrationExecutionMode { Local, Remote }` |
| `dukk/crates/ai/src/agent/action/…` | 🟡 Converte para o proto (`api::orchestration_config::ExecutionMode::Remote`) | existe conversão mas o runtime usa o type nativo |
| `dukk-code/dukk-core/crates/server` (`runtime_bridge.rs`, `toon-harness`, `compat-harness`) | 🔴 Harness próprios | `Harness` do proto (oz, claude_code, opencode, gemini, codex) não é referenciado |
| `dukk-proto-apis` `orchestration.proto` | ✅ Fonte da verdade | — |

### 2.3 `filters/v1` (AgentManagementFilters, StatusFilter, SourceFilter, HarnessFilter…)

| Onde | Situação | Evidência |
|---|---|---|
| `dukk/app/src/ai/agent_conversations_model/filter_proto.rs` | 🟡 ÚNICO uso real em runtime — mas só converte; runtime não consome | docstring: *"nothing in the runtime path uses the proto yet (this is the extraction/centralization step)"* |
| `dukk/app/src/ai/agent_conversations_model.rs` | 🟡 mantém enums idiomatic e converte no boundary | — |
| `dukk-code/dukk-core/crates/server` (rotas de agentes/runs) | 🔴 NÃO consome os filtros protobuf | sem nenhuma ref a `StatusFilter`/`AgentManagementFilters`/`SourceFilter` |
| `dukk-code/hermes-agent` (Python) | 🔴 não usa os `*_pb2.py` | sem imports de protobuf do contrato |

> 🎯 O contrato de filtros nasceu exatamente para o backend poder filtrar com o mesmo vocabulário do desktop — mas o backend ainda não usa (o próprio código diz "eventually").

### 2.4 `task.proto` (Task, Message, AgentEvent, ClientAction, ToolType…)

| Onde | Situação | Evidência |
|---|---|---|
| `dukk/app/src/integration_testing/agent_mode/*` | 🟡 Usa `dukk_multi_agent_api` **só em testes** (`step.rs`, `assertions.rs`, `llm_judge/mod.rs`) | decode de `Request`, `Message`, `GrepResult` |
| `dukk/crates/ai` | 🔴 tipo nativo de mensagens/conversa (modelo próprio) | não usa os bindings (só `prost-types`) |
| `dukk-code/hermes-agent` (Python) | 🔴 define mensagens/fluxo do agente à mão | sem `_pb2` |
| `dukk-code/dukk-core/crates/{flow,tools,runtime,api,server,…}` | 🔴 0 refs ao proto | todos os crates do workspace com 0 dependências protobuf |

### 2.5 `SISTEMA-INTERNO` (CRM — Go)

| Área | Situação | Evidência |
|---|---|---|
| `backend/go.mod` | 🔴 NÃO importa os bindings Go (`github.com/warpdotdev/warp-proto-apis/...` nem `nextlw`) | sem refs no `go.mod`/`go.sum` |
| `internal/dukk/client.go` | 🔴 structs JSON na mão (REST do dukk) | `TranscricaoResult`, `EscopoContent`, `RoadmapItem`, `EnvironmentInfo`… (linhas 27–737) |
| `internal/dukkcatalog/handler.go` | 🔴 proxy REST puro para `/v1/admin/*` do dukk-server | "o CRM NÃO fala com o Zitadel nem com o banco do dukk — só com a API" |
| `internal/tasks/model.go` | 🟡 `Task` própria (modelo de banco do CRM) | domínio do CRM — ok manter, MAS se trafegar com o dukk deveria converter no boundary |

> 📌 O CRM é Go e o proto-apis tem bindings Go **versionados** (`apis/multi_agent/v1/gen/go/*.pb.go`) — hoje o único consumidor Go do contrato seria o próprio repo (não há código Go fora que importe). O CRM poderia usar os bindings para tipar as respostas que vêm do dukk-server quando ele expuser protobuf/protojson.

---

## 3. Quadro-resumo de prioridades (recomendado)

| # | Prioridade | Ação | Repos afetados |
|---|---|---|---|
| 1 | 🔴 Alta | Unificar Kanban: eliminar `dukk-core/crates/kanban/src/schema.rs` e `dukk/app/src/kanban/types.rs` em favor do `kanban.proto` (bindings Rust). Adicionar conversão no boundary. | dukk-code, dukk |
| 2 | 🔴 Alta | `server/routes/agent_runs.rs`: usar o contrato de status em vez do `match` de string ("estado do contrato Warp"). | dukk-code |
| 3 | 🟠 Média | `filters/v1` no backend: servidor começa a consumir `AgentManagementFilters` (o próprio proto foi desenhado pra isso). | dukk-code |
| 4 | 🟠 Média | Desktop: `orchestration_config.rs` — manter type nativo se quiser, mas a conversão proto ↔ nativo deve ser a ÚNICA via de serialização (já existe em parte). | dukk |
| 5 | 🟡 Baixa | hermes-agent (Python): consumir os `*_pb2.py` para as mensagens do agente que já existem no contrato. | dukk-code |
| 6 | 🟡 Baixa | SISTEMA-INTERNO (Go): avaliar tipar com `gen/go` as respostas do dukk-server; começar pelos endpoints de kanban/runs quando o server expuser protojson. | SISTEMA-INTERNO |

---

## 4. Método

- Varredura de imports: `grep` por `warp_multi_agent_api`, `dukk_filters_api`, `warp-proto-apis`, `dukk-proto-apis`, `prost`, `protobuf`, `_pb2` nos três repos.
- Varredura de duplicação: busca por `struct/class KanbanBoard|KanbanCard|OrchestrationConfig|AgentManagementFilters`, `enum KanbanCardStatus`, `ToolType`, `ClientAction`.
- Exclusões: `.sandbox-home`, `.claude/worktrees`, `target/`, `node_modules/`, `.venv`.
- Falsos positivos ignorados (ex.: `session-sharing-protocol` em cache de git).
