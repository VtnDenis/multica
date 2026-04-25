# Agent Flag Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Add a repeatable `--agent` flag to `multica daemon start` that filters which agent CLIs are detected and registered at startup.

**Architecture:** Two-file change: add the flag definition and forwarding in the CLI command layer (`cmd_daemon.go`), add the filter field to the `Overrides` struct and apply filtering after auto-detection in the config layer (`config.go`).

**Tech Stack:** Go, Cobra CLI, existing daemon config system.

---

### Task 1: Add `--agent` flag definition to daemon start and restart commands

**Files:**
- Modify: `server/cmd/multica/cmd_daemon.go:59-91` (the `init()` function)

- [ ] **Step 1: Add the flag to `daemonStartCmd` in `init()`**

Add a `StringSlice` flag after the existing `--max-concurrent-tasks` flag (line 68):

```go
f.StringSlice("agent", nil, "Filter detected agents to this list (repeatable, e.g. --agent hermes --agent codex)")
```

Insert this line after line 68 (`f.Int("max-concurrent-tasks", 0, ...)`).

- [ ] **Step 2: Add the same flag to `daemonRestartCmd` in `init()`**

The restart command shares all the same flags as start. Add the same flag after line 84:

```go
rf.StringSlice("agent", nil, "Filter detected agents to this list (repeatable, e.g. --agent hermes --agent codex)")
```

Insert this line after line 84 (`rf.Int("max-concurrent-tasks", 0, ...)`).

- [ ] **Step 3: Run lsp_diagnostics on the modified file**

Run: `lsp_diagnostics` on `server/cmd/multica/cmd_daemon.go`
Expected: No errors

- [ ] **Step 4: Commit**

```bash
git add server/cmd/multica/cmd_daemon.go
git commit -m "feat: add --agent flag definition to daemon start/restart commands"
```

---

### Task 2: Add `OnlyAgents` field to `Overrides` struct and implement filtering logic

**Files:**
- Modify: `server/internal/daemon/config.go:54-68` (Overrides struct)
- Modify: `server/internal/daemon/config.go:84-157` (agent detection block)

- [ ] **Step 1: Add `OnlyAgents` field to `Overrides` struct**

Add after line 67 (`HealthPort int`):

```go
	OnlyAgents         []string // if non-empty, only these agents are detected
```

The struct should now look like:

```go
type Overrides struct {
	ServerURL          string
	WorkspacesRoot     string
	PollInterval       time.Duration
	HeartbeatInterval  time.Duration
	AgentTimeout       time.Duration
	MaxConcurrentTasks int
	DaemonID           string
	DeviceName         string
	RuntimeName        string
	Profile            string // profile name (empty = default)
	HealthPort         int    // health check port (0 = use default)
	OnlyAgents         []string // if non-empty, only these agents are detected
}
```

- [ ] **Step 2: Add filtering logic after the agent detection block**

Insert after line 154 (after the `kimi` detection block closes, before the `len(agents) == 0` check on line 155):

```go
	// Filter agents if OnlyAgents is specified
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

This filtering happens AFTER all agents are detected, so the existing `len(agents) == 0` error check on line 155 will correctly fail if the user specified agents that aren't installed.

- [ ] **Step 3: Run lsp_diagnostics on the modified file**

Run: `lsp_diagnostics` on `server/internal/daemon/config.go`
Expected: No errors

- [ ] **Step 4: Commit**

```bash
git add server/internal/daemon/config.go
git commit -m "feat: add OnlyAgents override and filtering logic to config"
```

---

### Task 3: Read `--agent` flag in `runDaemonForeground` and forward in `buildDaemonStartArgs`

**Files:**
- Modify: `server/cmd/multica/cmd_daemon.go:259-287` (runDaemonForeground function)
- Modify: `server/cmd/multica/cmd_daemon.go:222-257` (buildDaemonStartArgs function)

- [ ] **Step 1: Read `--agent` flag in `runDaemonForeground()`**

Add after line 287 (after the `max-concurrent-tasks` override block, before `cfg, err := daemon.LoadConfig(overrides)`):

```go
	if agents, _ := cmd.Flags().GetStringSlice("agent"); len(agents) > 0 {
		overrides.OnlyAgents = agents
	}
```

The code around line 285-290 should now look like:

```go
	if n, _ := cmd.Flags().GetInt("max-concurrent-tasks"); n > 0 {
		overrides.MaxConcurrentTasks = n
	}

	if agents, _ := cmd.Flags().GetStringSlice("agent"); len(agents) > 0 {
		overrides.OnlyAgents = agents
	}

	cfg, err := daemon.LoadConfig(overrides)
```

- [ ] **Step 2: Forward `--agent` flag in `buildDaemonStartArgs()`**

Add after line 246 (after the `max-concurrent-tasks` forwarding block, before the "Forward global persistent flags" comment):

```go
	if agents, _ := cmd.Flags().GetStringSlice("agent"); len(agents) > 0 {
		for _, a := range agents {
			args = append(args, "--agent", a)
		}
	}
```

The code around line 244-250 should now look like:

```go
	if n, _ := cmd.Flags().GetInt("max-concurrent-tasks"); n > 0 {
		args = append(args, "--max-concurrent-tasks", strconv.Itoa(n))
	}

	if agents, _ := cmd.Flags().GetStringSlice("agent"); len(agents) > 0 {
		for _, a := range agents {
			args = append(args, "--agent", a)
		}
	}

	// Forward global persistent flags.
```

- [ ] **Step 3: Run lsp_diagnostics on the modified file**

Run: `lsp_diagnostics` on `server/cmd/multica/cmd_daemon.go`
Expected: No errors

- [ ] **Step 4: Commit**

```bash
git add server/cmd/multica/cmd_daemon.go
git commit -m "feat: read and forward --agent flag in daemon start flow"
```

---

### Task 4: Verification

**Files:**
- All modified files

- [ ] **Step 1: Run full build check**

Run: `make check`
Expected: All checks pass (lint, build, tests)

- [ ] **Step 2: Verify help text shows the new flag**

Run: `cd server && go run ./cmd/multica daemon start --help`
Expected: Output includes `--agent` flag description:
```
      --agent strings   Filter detected agents to this list (repeatable, e.g. --agent hermes --agent codex)
```

- [ ] **Step 3: Final commit if any verification changes needed**

If any issues were found and fixed in the above steps, commit them:

```bash
git add -A
git commit -m "fix: address verification findings for --agent flag"
```
