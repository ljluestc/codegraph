# PR: Add optional PII sanitization layer to MCP output (`issue #383`)

## Summary

This PR adds an optional sanitization layer in the MCP response pipeline so CodeGraph can redact sensitive content before indexed code/context is returned to downstream LLM agents.

It introduces:
- A built-in sanitizer (`--sanitize`) for common PII/secrets.
- A custom sanitizer middleware hook (`--sanitize-hook <modulePath>`) for organization-specific policies and external integrations.
- Centralized response sanitization in the MCP tool execution path so all tool payloads are covered consistently.

## Problem

Issue #383 highlights a privacy gap: indexed code can contain sensitive values (emails, phone numbers, identifiers, secret-like tokens), and CodeGraph currently returns tool output directly to the MCP client/agent.

In regulated/compliance-sensitive environments, users need an explicit pre-LLM sanitization control without giving up MCP functionality.

## Solution

### 1) Built-in sanitization middleware

New module: `src/mcp/sanitization.ts`

Built-in redaction coverage:

| Category | Replacement token | False-positive guards |
| --- | --- | --- |
| Email addresses | `[REDACTED_EMAIL]` | — |
| Phone numbers | `[REDACTED_PHONE]` | Only redacts when the match contains 10–15 digits |
| US SSNs | `[REDACTED_SSN]` | — |
| Credit-card-like values | `[REDACTED_CREDIT_CARD]` | 13–19 digits **and** passes Luhn validation |
| OpenAI-style API keys (`sk-...`) | `[REDACTED_API_KEY]` | — |
| AWS access key IDs (`AKIA...`) | `[REDACTED_API_KEY]` | — |

The sanitizer is env-gated: it only runs when `CODEGRAPH_SANITIZE` is set to a truthy value (`1`, `true`, `yes`, `on`). When disabled, output is returned untouched.

### 2) Custom sanitizer hook

- Supports `CODEGRAPH_SANITIZE_HOOK` (or CLI `--sanitize-hook`) to load a user-provided module (absolute path, or resolved against the CWD).
- Hook contract: the module must export a function (default export or named `sanitize`) with signature `(text: string) => string | Promise<string>`.
- Runs in the MCP response pipeline after the built-in pass, so organizations can layer their own policy controls or external sanitizers on top.
- The hook spec and loaded module are cached per spec; load failures and runtime errors are logged once to stderr (no noisy repeat logging, no crash of the tool response).

### 3) Wired into the MCP tool output path

- Updated `ToolHandler.execute()` in `src/mcp/tools.ts`.
- Sanitization is applied once centrally to final tool results, after the existing wrappers (worktree / staleness notices), so response behavior remains consistent while enforcing redaction before output leaves MCP.
- `codegraph_status` is intentionally exempt from the wrapper changes (it embeds the pending-files list as a first-class section), and the sanitization pass preserves this behavior.

### 4) CLI and env controls

Updated `codegraph serve` in `src/bin/codegraph.ts`:

- `--sanitize` → sets `CODEGRAPH_SANITIZE=1`
- `--sanitize-hook <modulePath>` → sets `CODEGRAPH_SANITIZE_HOOK` (absolute or resolved path)

## Documentation updates

- `README.md` — new "MCP sanitization (optional)" section with CLI examples and env equivalents; `--sanitize` added to the command reference.
- `site/src/content/docs/reference/cli.md` — `--sanitize` / `--sanitize-hook` usage.
- `site/src/content/docs/reference/mcp-server.md` — optional privacy hardening section.
- `CHANGELOG.md` — entry under `[Unreleased]`.

## Tests

Added `__tests__/mcp-sanitization.test.ts` covering:
- Built-in sanitizer redaction behavior (including Luhn-gated credit cards and digit-length-gated phone numbers).
- Env-gated built-in sanitization in MCP tool responses.
- Disabled-mode passthrough (no redaction, output untouched).
- Custom hook execution in the MCP response path.

## Validation run

- `npm run build`
- `npx vitest run __tests__/mcp-sanitization.test.ts`
- `npx vitest run __tests__/mcp-tool-allowlist.test.ts`

## Backward compatibility

- Default behavior is unchanged unless sanitization is explicitly enabled.
- Existing MCP workflows continue to function as-is; sanitization composes with (does not replace) the existing worktree/staleness notices.

## Issue linkage

- Closes #383
