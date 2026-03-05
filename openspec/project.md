# Project Context

## Purpose

**Cline** is an AI coding assistant that works in your **CLI** a**N**d **E**ditor. It provides a human-in-the-loop experience: an agentic AI that can create and edit files, run terminal commands, use the browser (e.g. for testing), and extend itself via the Model Context Protocol (MCP), while requiring user approval for file changes and terminal commands.

Goals:
- Assist with complex software development tasks step-by-step using Claude and other models.
- Integrate with the IDE (VS Code extension) and a standalone CLI (React Ink TUI).
- Support many API providers (Anthropic, OpenAI, OpenRouter, Google Gemini, AWS Bedrock, Azure, GCP Vertex, Cerebras, Groq, LM Studio, Ollama, etc.).
- Let users add custom tools via MCP and “add a tool that…” flows.
- Track tokens and API usage; work behind corporate proxies; support multiple platforms (VS Code, JetBrains, CLI).

## Tech Stack

- **Runtime**: Node.js, TypeScript (strict mode, ES2022 target)
- **Extension**: VS Code Extension API (engines `^1.84.0`), entry `dist/extension.js`
- **Build**: esbuild (extension bundle), Vite (webview), `npm run compile` (not `build`)
- **Webview**: React, Vite, Tailwind CSS v4
- **CLI**: React Ink (terminal UI), in `cli/`
- **Communication**: gRPC-like protocol over VS Code message passing; Protobuf definitions in `proto/`, generated code in `src/generated/`
- **Lint/format**: Biome (lint + format); ESLint/Prettier not used
- **Testing**: Mocha (unit), VS Code test runner (integration), Playwright (e2e)
- **Package manager**: npm, workspaces for root and `cli`

## Project Conventions

### Code Style

- **Formatter**: Biome. Use `npm run format:fix` before PRs; lint-staged runs `biome check --write --staged` on commit.
- **Format**: Tabs, width 4, line width 130, LF; semicolons as needed; double quotes for JSX; trailing commas.
- **Imports**: Organize with Biome (`source.organizeImports`). Use path aliases: `@/*`, `@api/*`, `@core/*`, `@generated/*`, `@hosts/*`, etc. (see `tsconfig.json`).
- **Logging**: Use the Logger service; avoid raw `console` in extension code (grit rule enforces this).
- **Naming**: Proto services `PascalCaseService`, RPCs `camelCase`, messages `PascalCase`. Capabilities/specs: verb-noun, kebab-case (e.g. `user-auth`).

### Architecture Patterns

- **Extension ↔ Webview**: Communicate via gRPC over VS Code messaging. Proto files per domain (`proto/cline/*.proto`). Run `npm run protos` after proto changes; generates types in `src/shared/proto/`, implementations in `src/generated/grpc-js/`, `src/generated/nice-grpc/`, `src/generated/hosts/`.
- **State**: Global state keys in `src/shared/storage/state-keys.ts`; read in `src/core/storage/utils/state-helpers.ts` (including `context.globalState.get()` in `readGlobalStateFromDisk()`). Use `StateManager` for normal read/write; for cross-window reads at startup (before cache ready), read from `context.globalState.get()` in `common.ts`.
- **API providers**: Register in proto (`proto/cline/models.proto`), `src/shared/proto-conversions/models/api-configuration-conversion.ts` (both directions), `src/shared/api.ts`, `src/core/api/index.ts`, webview provider list and validation. CLI: `cli/src/components/ModelPicker.tsx` and provider config utils.
- **System prompt**: Modular components + variants (model-specific) + templates. See `src/core/prompts/system-prompt/README.md` and `tools/README.md`. New tools: add to `ClineDefaultTool`, define in `src/core/prompts/system-prompt/tools/`, register in `init.ts`, add to variant configs, add handler and any UI (e.g. `ClineSay` in proto + conversions + `ChatRow.tsx`).
- **Networking**: In extension code, never use global `fetch` or default axios; use `@/shared/net` (`fetch` wrapper and `getAxiosSettings()`) for proxy support. In webview, global `fetch` is fine.

### Testing Strategy

- **Unit**: Mocha, `npm run test:unit` (uses `tsconfig.unit-test.json`). System prompt tests use snapshots; regenerate with `UPDATE_SNAPSHOTS=true npm run test:unit`.
- **Integration**: VS Code test runner, `npm run test:integration`.
- **E2E**: Playwright; `npm run test:e2e` (builds VSIX and runs Playwright).
- **CI**: `npm run test` runs unit + integration. Run `npm run format:fix` and `npm run lint` before submitting PRs.
- **Coverage**: c8; see `package.json` c8 config for excludes (e.g. webview-ui, generated, tests).

### Git Workflow

- **Default branch**: `main`.
- **Features**: Start from a GitHub Issue; for non-trivial features use the Feature Requests discussions board and get maintainer approval before implementation. PRs without an approved issue may be closed.
- **Commits**: Normal feature/bugfix branches; no contributor-written changelog entries (maintainers handle release/changelog).
- **PRs**: Use the pull request template; link the related issue; run tests and `npm run format:fix`; keep changes focused (single feature/bugfix/chore).
- **OpenSpec**: Use `openspec/` for spec-driven changes: `openspec list` / `openspec list --specs`, create proposals under `changes/<change-id>/` with `proposal.md`, `tasks.md`, optional `design.md`, and spec deltas; validate with `openspec validate <id> --strict`. See `openspec/AGENTS.md` for the full workflow.

## Domain Context

- **Cline** is the product name; the extension package is `claude-dev`, display name “Cline”.
- **Tasks**: User opens a “task” (chat session); the model uses tools (read/edit files, run commands, browser, MCP) and the UI shows diffs and approvals.
- **Model families**: System prompt and tools have variants (e.g. generic, next-gen, xs, native-gpt-5); tool specs and variant configs must stay in sync.
- **Cancellation**: When a task is cancelled mid-stream, message status may stay “generating”; UI should detect cancellation via `!isLast` or `lastModifiedMessage?.ask === "resume_task" | "resume_completed_task"` and backend should check `taskState.abort` for cleanup.
- **Responses API (e.g. OpenAI Codex)**: Uses native tool calling; provider must be in `isNextGenModelProvider()` and models must have `apiFormat: ApiFormat.OPENAI_RESPONSES` so native tools are used.

## Important Constraints

- **VS Code**: This is a VS Code extension; use `npm run compile` (and related scripts in `package.json`) for builds.
- **Proto round-trip**: API provider and other enums must be mirrored in proto and conversion layer (to/from proto); missing mappings can silently reset to defaults (e.g. Anthropic).
- **State and settings**: New global state keys need: type in `state-keys.ts`, read in `state-helpers.ts` (including `globalState.get`), and for user-facing settings also webview paths (`UpdateSettingsRequest`, `getStateToPostToWebview`, ExtensionState/webview defaults) and both update paths (webview and CLI/ACP).
- **JetBrains/CLI**: Network calls must use proxy-aware fetch/settings from `@/shared/net` so they work in all environments.
- **Simplicity**: Prefer small, focused changes; avoid unnecessary frameworks or abstraction until justified by scale or multiple use cases.

## External Dependencies

- **AI APIs**: Anthropic, OpenAI, OpenRouter, Google (Vertex, Gemini), AWS Bedrock, Azure, Cerebras, Groq, Mistral, and others; see `src/shared/api.ts` and `src/core/api/`.
- **MCP**: Model Context Protocol SDK for custom tools and servers.
- **Observability**: OpenTelemetry (traces, metrics, logs).
- **Docs**: Hosted at docs.cline.bot; repo has `docs/` with its own tooling.
- **Evals**: Smoke and e2e evals in `evals/`; optional `evals.env` for activation.
