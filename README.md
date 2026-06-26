# warp-proto-apis

Repository to centralize protobuf-based APIs for Warp services and clients.
Maintains proto definitions alongside generated code for supported clients.

## General structure
```
warp-server-apis/
└── apis/
    └── <api>/
        └── <version>/
            ├── <api>.proto // or could be broken down into multiple .proto files
            └── gen/
                └── <bindings_for_lang>/
```

## APIs

| API | Package | Edition/Syntax | Bindings |
| --- | --- | --- | --- |
| `multi_agent/v1` | `warp.multi_agent.v1` | edition 2023 | Go (checked in) + Rust (`warp_multi_agent_api`) |
| `filters/v1` | `dukk.filters.v1` | proto3 | Rust only (`dukk_filters_api`) |

None of the APIs currently define gRPC `service` blocks; the `.proto` files
describe the request/response/data **messages and enums** exchanged between
clients and services.

### `multi_agent/v1`
The primary API for Warp's multi-agent system, broken down into multiple
`.proto` files (all in package `warp.multi_agent.v1`):

- `request.proto` — `Request` plus the `AutonomyLevel` and `IsolationLevel` enums.
- `response.proto` — `ResponseEvent`, `ClientAction`, and the `LLMProvider` enum.
- `task.proto` — `Task`, `Message`, `AgentEvent`, the full set of tool-call result
  messages (e.g. `RunShellCommandResult`, `ReadFilesResult`, `ApplyFileDiffsResult`,
  `CallMCPToolResult`, `DukkToolResult`, `UseComputerResult`, …), the
  sub-agent orchestration messages (`StartAgent`, `StartAgentV2`, `RunAgents`,
  `SendMessageToAgent`, `AskUserQuestion`, …), and the enums `LifecycleEventType`,
  `ToolType`, `AgentType`, `RiskCategory`, `UserQueryMode`.
- `kanban.proto` — `KanbanBoard`, `KanbanCard`, `KanbanRun`, `KanbanLink`,
  `KanbanComment`, `KanbanEvent` plus the `KanbanCardStatus` and `KanbanEventKind` enums.
- `orchestration.proto` — `Harness`, `OrchestrationConfig`, `OrchestrationStatus`,
  `OrchestrationConfigUpdate`, `OrchestrationConfigSnapshot`.
- `skill.proto` — `SkillDescriptor`, `SkillRef`, `Skill`.
- `lsp.proto` — `LspDescriptor`.
- `input_context.proto` — `InputContext`.
- `attachment.proto` — `Attachment` and the attachable object messages
  (`ExecutedShellCommand`, `RunningShellCommand`, `DriveObject`, `Workflow`,
  `Notebook`, `DiffHunk`, `DiffSet`, `FilePathReference`, …).
- `file_content.proto` — `FileContent`, `BinaryFileContent`, `AnyFileContent`,
  `FileContentLineRange`.
- `document_content.proto` — `DocumentContent`.
- `conversation_data.proto` — `ConversationData`.
- `citations.proto` — `Citation` plus the `DocumentType` enum.
- `suggestions.proto` — `Suggestions`, `SuggestedRule`, `SuggestedAgentModeWorkflow`.
- `todo.proto` — `TodoItem`, `CreateTodoList`, `UpdatePendingTodos`, `MarkTodosCompleted`.
- `options.proto` — custom field options only (`(sensitive)`, `(internal)`); defines no messages.

### `filters/v1`
Agent-management filtering API (package `dukk.filters.v1`, `filters.proto`):

- Messages: `AgentManagementFilters`, `SourceFilter`, `CreatorRef`, `CreatorFilter`,
  `EnvironmentFilter`, `HarnessFilter`.
- Enums: `StatusFilter`, `ArtifactFilter`, `CreatedOnFilter`, `OwnerFilter`,
  `AgentSource`, `EnvironmentFilterMode`.

This API ships **Rust bindings only** (crate `dukk_filters_api`). It has no Go
module and is not handled by `./script/generate` (see below).

## Initial setup

Run `./script/bootstrap` to install proto compiler dependencies.

## Updating generated bindings

When updating the proto definitions, you will need to run the `./script/generate` script.  This will automatically update the **Go** bindings.

For example, to update the `multi_agent` API:
```bash
./script/generate -a multi_agent -v v1
```

> Note: `./script/generate` only generates Go bindings, and therefore applies
> only to APIs that have a Go module (currently `multi_agent`). The `filters`
> API has no Go module and is Rust-only — its bindings are generated at compile
> time by `cargo build` and nothing needs to be regenerated for it.

## Required dependencies
Must have `protoc` installed. See here on how to install for your platform: https://protobuf.dev/installation/.

### Go
Requires the `protoc-gen-go` plugin: `go install google.golang.org/protobuf/cmd/protoc-gen-go@latest`.

This is installed by the bootstrap script.

The generated Go bindings for `multi_agent/v1` are checked into
`apis/multi_agent/v1/gen/go/` and imported via
`github.com/warpdotdev/warp-proto-apis/apis/multi_agent/v1/gen/go`.

### Rust
There are no specific dependencies required for Rust, outside of the `protoc` compiler and a Rust toolchain.  The Rust code generation happens at compile time (as part of a Rust build script), so no additional setup is required and nothing needs to be regenerated and checked in when proto files are modified.

The Rust workspace (root `Cargo.toml`) builds one crate per API:
`warp_multi_agent_api` and `dukk_filters_api`.

## License

This project is licensed under version 3 of the GNU Affero General Public License; see LICENSE.md.

Warp requires contributors to sign a contributor license agreement (CLA) before their contributions can be merged. You can read and sign our CLA at https://cla.warp.dev.
</content>
</invoke>
