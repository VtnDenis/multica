# Design: `--agent` Flag for Daemon Start

**Date:** 2026-04-25
**Author:** Sisyphus
**Status:** Draft — pending user review

## Problem

The daemon auto-detects all agent CLIs on PATH (claude, codex, opencode, openclaw, hermes, gemini, pi, cursor, copilot, kimi). Users cannot restrict which agents are registered at startup. There is no CLI flag or config option to filter detected agents.

## Solution

Add a repeatable `--agent` flag to `multica daemon start` that acts as a whitelist. When provided, only the specified agents are registered. When omitted, behavior is unchanged (auto-detect all).

### CLI Interface

```bash
multica daemon start --agent hermes                    # Only Hermes
multica daemon start --agent hermes --agent codex      # Hermes + Codex
multica daemon start                                    # Auto-detect all (existing behavior)
```

### Files Modified

| File | Change |
|------|--------|
| `server/cmd/multica/cmd_daemon.go` | Add `--agent` flag in `init()`, read in `runDaemonForeground()`, forward in `buildDaemonStartArgs()` |
| `server/internal/daemon/config.go` | Add `OnlyAgents []string` to `Overrides`, filter agents map after detection |

### Detailed Changes

#### 1. `server/cmd/multica/cmd_daemon.go`

**Flag definition** (in `init()`, alongside existing flags):
```go
f.StringSlice("agent", nil, "Filter detected agents to this list (repeatable, e.g. --agent hermes --agent codex)")
```

**Read flag in `runDaemonForeground()`** (after existing flag reads, before `LoadConfig`):
```go
if agents, _ := cmd.Flags().GetStringSlice("agent"); len(agents) > 0 {
    overrides.OnlyAgents = agents
}
```

**Forward in `buildDaemonStartArgs()`** (after existing flag forwarding):
```go
if agents, _ := cmd.Flags().GetStringSlice("agent"); len(agents) > 0 {
    for _, a := range agents {
        args = append(args, "--agent", a)
    }
}
```

#### 2. `server/internal/daemon/config.go`

**Add to `Overrides` struct** (after existing fields):
```go
OnlyAgents []string // if non-empty, only these agents are detected
```

**Filter logic** (after the full agents map is built, ~line 154, before the `len(agents) == 0` check):
```go
if len(overrides.OnlyAgents) > 0 {
    filtered := map[string]AgentEntry{}
    for _, name := range overrides.OnlyAgents {
        if entry, ok := agents[name]; ok {
            filtered[name] = entry
        }
    }
    agents = filtered
}
```

### Error Handling

- If `--agent unknown-provider` is passed and that CLI isn't found on PATH, the filtered map will be empty and the daemon fails with the existing "no agents detected" error. This is correct fail-fast behavior.
- No explicit validation of provider names needed — the detection loop handles unknown names gracefully (they simply won't exist in the map).

### Backward Compatibility

- Fully backward compatible. The flag is optional and defaults to nil/empty.
- Existing `multica daemon start` behavior is unchanged when the flag is not provided.
- No changes to config file format, API, or daemon registration logic.

### Testing

- `lsp_diagnostics` clean on modified files
- `make check` passes
- Manual: `multica daemon start --agent hermes` should only report Hermes as available in the UI
