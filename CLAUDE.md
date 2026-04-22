# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Build Commands

```bash
npm run build          # Compile TypeScript + copy assets + build manifest
npm run dev            # Run from source with tsx (no build needed)
npm run dev:bun        # Run from source with bun
npm start              # Run built CLI from dist/
npm run typecheck      # Type-check without compiling
```

## Test Commands

```bash
npm test               # Run unit + extension + adapter tests (default)
npm run test:adapter   # Run only adapter tests
npm run test:e2e       # Run E2E tests (requires built dist/)
npm run test:all       # Run all test suites including e2e/smoke

# Single file
npx vitest run src/cli.test.ts

# Watch mode during development
npx vitest src/
```

## Architecture

### Directory Layout

```
src/
  main.ts          # CLI entry point, fast paths for --version/completion
  cli.ts           # Commander setup, built-in commands (list/validate/browser/etc)
  commanderAdapter.ts  # Wires dynamic adapter commands into Commander
  discovery.ts     # Finds and loads adapters from clis/ + ~/.opencli/clis/
  registry.ts     # Command registration and lookup
  pipeline/       # Execution pipeline (executor, template, steps)
  browser/        # Browser Bridge via CDP (page.ts, cdp.ts, bridge.ts)
  commands/       # Daemon and other sub-commands
  download/       # Media and article download logic
clis/             # Adapter commands (JavaScript-first, not TypeScript)
extension/        # Browser Bridge Chrome extension
tests/
  e2e/           # End-to-end CLI tests
  smoke/         # API health and manifest validation
```

### Key Concepts

- **Browser Bridge**: CLI connects to running Chrome via CDP through a Browser Bridge extension. The daemon (`src/daemon.ts`) bridges extension ↔ CLI. Configure with `OPENCLI_DAEMON_PORT` (default 19825).

- **Adapters** (`clis/`): Site-specific commands written in JavaScript. They export a `browser: boolean` flag and a `run()` function. User adapters live in `~/.opencli/clis/` and override built-in ones.

- **Pipeline** (`src/pipeline/`): Execution model for commands. Steps include `fetch`, `browser`, `intercept`, `transform`, `tap`, `download`. Template syntax `{{variable}}` resolves from context.

- **CLI Hub**: `opencli` acts as a dispatcher for external tools (`gh`, `docker`, etc.) defined in `src/external-clis.yaml`. Use `opencli register` to add local binaries.

- **Output formats**: All built-in commands support `--format` (`table`, `json`, `yaml`, `md`, `csv`) via `src/output.ts`.

### Startup Flow

1. `main.ts` handles fast paths (--version, completion) first to avoid startup cost
2. Loads discovery, hooks, node-network in parallel
3. Discovers built-in adapters from `clis/`, then user adapters from `~/.opencli/clis/`, then plugins
4. User adapters override built-in ones (same name wins)
5. `runCli()` in `cli.ts` sets up Commander and enters command loop

### Exit Codes

| Code | Meaning |
|------|---------|
| 0 | Success |
| 1 | Generic error |
| 2 | Usage error |
| 66 | Empty result (EX_NOINPUT) |
| 69 | Browser Bridge not connected (EX_UNAVAILABLE) |
| 75 | Timeout (EX_TEMPFAIL) |
| 77 | Auth required (EX_NOPERM) |

### Environment Variables

| Variable | Default | Purpose |
|----------|---------|---------|
| `OPENCLI_DAEMON_PORT` | 19825 | Daemon-extension bridge port |
| `OPENCLI_CDP_ENDPOINT` | — | Chrome DevTools Protocol endpoint |
| `OPENCLI_CDP_TARGET` | — | Filter CDP targets by URL substring |
| `OPENCLI_BROWSER_COMMAND_TIMEOUT` | 60 | Seconds to wait for browser command |
| `OPENCLI_VERBOSE` | false | Enable verbose logging (-v flag also works) |