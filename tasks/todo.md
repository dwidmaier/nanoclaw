# Nanoclaw Alpha — Task Tracker

See [docs/PRD.md](../docs/PRD.md) for the full PRD.

## Phase 1 — Prove It Works

- [ ] M1: Nanoclaw running on Linux box (Docker, Node.js 22, Tailscale)
- [ ] M2: Hono HTTP API accepting task submissions from Nessie over Tailscale
- [ ] M3: Plainjob SQLite queue managing task lifecycle
- [ ] M4: Containerized agent execution via existing container-runner
- [ ] M5: Result reporting back to Nessie via HTTP callback
- [ ] M6: Discord notifications: task started, completed, failed, errors
- [ ] M7: Basic health endpoint (`GET /health`)
- [ ] M8: Graceful shutdown (finish active containers on SIGTERM)
- [ ] M9: 10 proven task types at >80% success rate
- [ ] M10: Langfuse integration for agent trace collection

## Phase 1b — Self-Improvement

- [ ] M11: Automatic post-mortem on every failure
- [ ] M12: Post-mortems written to `groups/{group}/CLAUDE.md`
- [ ] M13: Weekly retrospective task (scheduled)
- [ ] M14: Success rate tracking per task type in Langfuse
