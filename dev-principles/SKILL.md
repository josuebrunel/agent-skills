---
name: dev-principles
description: Josue's core software development principles — idiomatic code, explicit error handling, security hygiene, DRY, testing discipline, structured logging, context/cancellation propagation, transaction handling, centralized error handling, config management, and standing up local dependencies via Docker Compose. Applies to any language or stack, not just Go — use this for any non-trivial coding task. Stack-specific skills (e.g. go-stack) build on top of this and add the concrete library/tool instantiation of these same principles.
---

# Josue's Development Principles

Cross-cutting principles that apply regardless of language or framework. These are the "why" — stack-specific skills (like `go-stack`) provide the "how" for a particular toolset. When both apply, follow the stack-specific skill's concrete instantiation; when no stack-specific skill covers something, apply these directly.

## Idiomatic, standards-compliant code
- Use the language's standard formatter and linter (e.g. `gofmt`/`prettier`/`black`) rather than hand-formatting or ignoring lint warnings.
- Follow the language/ecosystem's standard naming and documentation conventions instead of inventing your own style.
- Keep module/package boundaries clean (no import cycles, small focused units) rather than one large catch-all utility dump.

## Error handling
- Handle errors explicitly at the point they occur — wrap them with context about what was being attempted, don't swallow them silently.
- Don't use exceptions/panics/crashes for expected or recoverable failure conditions — reserve them for truly unrecoverable states.

## Security
- Never build queries via string concatenation — use parameterized queries/prepared statements or a query builder that binds parameters, the primary defense against injection.
- Validate/sanitize external input (HTTP params, form data, message payloads) at the boundary, before it reaches business logic.
- Never hardcode secrets, API keys, or credentials — load them from environment/config, and never log them.
- Enforce least-privilege authorization checks on every protected operation, not just at a coarse routing/gateway level.

## DRY
- Before writing a new helper, query, or component, search the codebase for an existing one that does the same thing — reuse or extend it instead of duplicating.
- Extract shared logic (validation, formatting, query fragments) into a common function/module once it's used in more than one place, instead of copy-pasting.
- Applies across every layer of the stack — business logic, data access, and UI/presentation alike.

## Testing
- New functionality and non-trivial helpers ship with tests in the same change, not as a follow-up.
- Prefer table-driven/data-driven tests where the language's testing tools support the pattern.
- Test files/modules live alongside the code they cover, matching the ecosystem's convention.

## Structured logging
- Use structured key-value logging instead of unstructured string concatenation — makes logs greppable and machine-parseable.
- Use consistent log levels (debug/info/warn/error) rather than logging everything at one level.
- Never log secrets, tokens, or full request/session payloads.

## Context and cancellation propagation
- Thread cancellation/timeout context through the call stack (request → business logic → downstream calls) rather than starting fresh context deep in the stack.
- Respect cancellation/timeouts from the originating caller so slow downstream calls don't outlive a client that's gone away.

## Transaction handling
- Wrap multi-step writes (create-then-update-related-record) in an explicit transaction with rollback on error, instead of issuing sequential unguarded writes that can leave partial state on failure.
- Keep transaction scope tight — open right before the first write, commit/rollback right after the last.

## Centralized error handling
- Use a single, centralized place that maps errors to a consistent response shape/status, instead of ad hoc error-to-response logic scattered at every call site.
- Propagate errors up to that central handler rather than writing the response directly on every error path.

## Config management
- Load configuration from environment into a single typed config structure once at startup, rather than scattered ad hoc lookups throughout the codebase.
- Fail fast at startup if required config is missing or invalid, instead of discovering it later at request/runtime.

## Local dependencies via Docker Compose
- Any project with local dependencies (database, cache, message queue, etc.) ships a `docker-compose.yml` at the repo root to stand them up — don't require manually-installed local services or undocumented ad hoc `docker run` commands.
- Create it when scaffolding the project, not as an afterthought. Add a service to it whenever the project gains a new local dependency, so `docker compose up` always reflects what the project actually needs.
- Exact services and wiring are stack-specific — e.g. see `go-stack` for its postgres + app service setup — this principle just requires that *some* compose file exists and stays current.

## When NOT to apply this
- If a project's existing conventions or linter configuration conflict with a point above, follow the project's established convention rather than forcing a change mid-task.
- If a more specific stack skill is active (e.g. `go-stack` for Go work), follow its concrete instantiation of these principles rather than re-deriving one from scratch.
