# Security Review: Agent Deck

**Review Date:** 2026-01-09
**Reviewed By:** Claude Opus 4.5
**Codebase Version:** Commit b5e0ab3 (main branch)

---

## Summary

This tool appears to be legitimate session management software for AI coding agents with **no malware, keyloggers, or malicious code**.

---

## What This Tool Does (Legitimate Purpose)

Agent Deck is a terminal-based session manager for AI coding agents (Claude Code, Gemini CLI, etc.) built on tmux. It provides:
- Session management and organization
- Status detection (running/waiting/idle)
- MCP (Model Context Protocol) server pooling
- Session forking for Claude

---

## Security Analysis Results

### No Keyloggers or Input Capture

- **PTY handling** (`internal/tmux/pty.go`): The only input interception is for **Ctrl+Q** (ASCII 17) to detach from sessions - standard tmux-like behavior
- Keyboard input is passed directly to tmux sessions, not logged or exfiltrated
- No suspicious input capture patterns found

### No Data Exfiltration

- **Network requests are limited to**:
  - GitHub API at `api.github.com/repos/asheshgoplani/agent-deck/releases/latest` for update checks
  - Binary downloads from GitHub releases
- No telemetry, analytics, or data upload endpoints
- No external server communication beyond GitHub

### No Hardcoded Secrets

- API keys/tokens in the codebase are only **examples in documentation** (README.md, config examples)
- No embedded credentials or private keys

### No Malicious Code Patterns

- **Clipboard access**: Only configures tmux for standard copy-paste integration - no reading or exfiltrating clipboard data
- **Command execution**: All `exec.Command` calls are for `tmux` operations (standard for a tmux manager)
- **File operations**: Only read/write to `~/.agent-deck/` directory with proper path traversal protection

---

## Positive Security Practices Found

| Practice | Location |
|----------|----------|
| **Path traversal protection** | `storage.go:21-50` - validates expanded paths stay under home directory |
| **Secure file permissions** | `0700` for directories, `0600` for session files |
| **Atomic writes** | Rolling backups + atomic rename pattern to prevent data corruption |
| **Lock files** | Prevents multiple instances with PID-based locking |
| **tmux `-l` flag** | Uses literal text mode for SendKeys to prevent command injection |

---

## Minor Security Considerations

### 1. Update Mechanism (`internal/update/update.go`)

- Downloads binaries from GitHub over HTTPS
- **Risk**: No cryptographic signature verification of downloads (relies on HTTPS/TLS)
- **Mitigation**: Standard for many Go tools; GitHub provides integrity

### 2. MCP Socket Proxy (`internal/mcppool/socket_proxy.go`)

- Creates Unix sockets in `/tmp/agentdeck-mcp-*.sock`
- **Risk**: Local socket permissions (standard Unix socket security)
- **Mitigation**: Sockets are created with default permissions; only local access

### 3. Shell Command Building (`internal/session/instance.go:186-254`)

- Builds Claude/Gemini commands with session IDs
- Single quotes used for message escaping: `strings.ReplaceAll(message, "'", "'\"'\"'")`
- **Risk**: Complex shell escaping, though patterns look correct

---

## Install Script Review (`install.sh`)

The install script:
- Downloads from `github.com/asheshgoplani/agent-deck` releases
- Optionally installs tmux/jq via standard package managers
- Configures `~/.tmux.conf` with mouse/clipboard settings
- **No suspicious behavior** - standard installer pattern

---

## Files Reviewed

### Core Application
- `cmd/agent-deck/main.go` - Entry point and CLI
- `internal/session/instance.go` - Session lifecycle management
- `internal/session/storage.go` - Data persistence
- `internal/tmux/tmux.go` - Tmux integration
- `internal/tmux/pty.go` - PTY handling for session attachment

### Network & External Communication
- `internal/update/update.go` - GitHub update checking (only network code)

### MCP Integration
- `internal/mcppool/socket_proxy.go` - Unix socket proxy for MCP servers

### Installation
- `install.sh` - Installation script

---

## Verdict

**This tool is safe to use.** It is a well-architected Go application with:
- No malware or keyloggers
- No data exfiltration
- No backdoors or hidden functionality
- Reasonable security practices for a CLI tool

The only outbound network connection is to GitHub for optional update checks, which is transparent and standard. All operations are local to your tmux sessions and `~/.agent-deck/` directory.

---

## Recommendations

1. **Consider adding binary signature verification** for the self-update feature (e.g., GPG signatures or cosign)
2. **Document the network behavior** in README to be transparent about GitHub API calls
3. **Consider restricting socket permissions** for MCP sockets in `/tmp/`

---

*This review was conducted by analyzing source code patterns, searching for security-sensitive operations, and examining all network, file I/O, and command execution paths.*
