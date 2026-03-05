# Nanoclaw Alpha — Lean MVP PRD (Revised)

## Context

Human attention is the bottleneck. This project builds a hierarchical autonomous agent organization: **Nessie** (OpenClaw on Mac Mini) is Chief of Staff, **Nanoclaw instances** are department heads on dedicated machines. The first deployment — **Nanoclaw Alpha** — specializes in Software Engineering on a spare Linux box.

The existing nanoclaw framework (forked to `dwidmaier/nanoclaw`) already handles 80% of what we need: containerized agent execution, Discord integration, SQLite, IPC, task scheduling, memory. The remaining 20% is a task input API, result reporting, health monitoring, and — critically — a self-improvement loop that makes agents get better over time.

**Design philosophy**: Integrate production-ready components, don't reinvent. Prove agent quality before building infrastructure. The bottleneck is agent intelligence, not plumbing.

---

## Vision

```
Dan (CEO)
  │
  ├── Dashboard (web UI on Nessie) ← monitoring, intervention, oversight
  ├── Discord notifications ← alerts, completions, errors from all agents
  │
  └── Nessie (Chief of Staff — OpenClaw on Mac Mini)
        │
        ├── Nanoclaw Alpha (Software Engineering — Linux box)
        │     └── Containerized agents (Claude Code SDK)
        ├── Nanoclaw Beta (future)
        └── Nanoclaw N (future)
```

**Discord** = agent-to-human communication channel. Each agent posts notifications, alerts, completions, and errors to Discord so Dan can stay informed without SSH-ing into machines. It is NOT a dashboard or control plane.

**Dashboard** = web UI hosted on Nessie. Shows real-time status of all agents, task queues, cost tracking, success rates, and allows intervention (cancel, reprioritize, redirect). This is Dan's primary interface for oversight.

---

## Architecture

```
Mac Mini (Nessie)                        Linux Box (Nanoclaw Alpha)
+----------------------------+           +------------------------------+
| OpenClaw / Nessie          |           | Nanoclaw Alpha               |
|                            |  HTTP     |                              |
| Dashboard (web UI) -------<==========>--- Hono API (task endpoint)   |
| (Tailscale: 100.x.x.1)   | over      | (Tailscale: 100.x.x.2)     |
|                            | Tailscale |                              |
| Langfuse (self-hosted)    <===========--- Trace push                 |
|                            |           |                              |
+----------------------------+           | SQLite job queue (Plainjob)  |
                                         | Docker Engine                |
Discord (cloud)                          | +-------+ +-------+         |
+----------------------------+           | |Agent 1| |Agent 2| ...     |
| #alpha-notifications      <===========--- Status posts               |
| #nessie-notifications     |           | +-------+ +-------+         |
| #costs                    |           +------------------------------+
+----------------------------+
```

### What's Already Built (Integrate, Don't Reinvent)

| Need | Existing Component | Status |
|------|-------------------|--------|
| Agent execution in containers | Nanoclaw `container-runner.ts` + `agent-runner/` | Production-ready, use as-is |
| Claude Code SDK integration | Nanoclaw `agent-runner/src/index.ts` → `query()` | Production-ready, use as-is |
| Concurrency control + retry | Nanoclaw `group-queue.ts` | Production-ready, use as-is |
| Container ↔ host IPC | Nanoclaw `ipc.ts` + `ipc-auth.ts` | Production-ready, use as-is |
| SQLite state management | Nanoclaw `db.ts` (better-sqlite3) | Production-ready, use as-is |
| Scheduled tasks (cron/interval) | Nanoclaw `task-scheduler.ts` | Production-ready, use as-is |
| Agent memory | Nanoclaw CLAUDE.md hierarchy | Production-ready, use as-is |
| Discord integration | Nanoclaw `add-discord` skill (discord.js) | Production-ready, apply skill |
| Container security model | Nanoclaw mount-security, non-root user, isolation | Production-ready, use as-is |
| Channel registry pattern | Nanoclaw `channels/registry.ts` | Production-ready, extend with HTTP channel |
| Message routing | Nanoclaw `router.ts` | Production-ready, use as-is |

### New Components to Integrate

| Need | Component | Why This One |
|------|-----------|-------------|
| HTTP task API | **Hono** | 3x faster than Express, TypeScript-native, ~50 lines for task endpoint |
| Job queue with retry/priority | **Plainjob** | Designed for better-sqlite3 (already a dep), 15k jobs/sec, retry + priority built-in |
| Agent observability | **Langfuse** (self-hosted on Nessie) | MIT-licensed, traces agent execution, failure clustering, success rate tracking |
| Dashboard | **Langfuse UI** + custom Hono dashboard | Langfuse provides agent traces; add a lightweight task status dashboard on Nessie |

### What We Don't Build

| Component | Why Not |
|-----------|---------|
| Redis infrastructure | Overkill for two machines. HTTP over Tailscale is simpler. Add Redis when we have 5+ Nanoclaws |
| Custom channel stripping | Channels with missing env vars are auto-skipped (existing behavior). Change nothing |
| Complex Discord bot commands | Discord is for notifications OUT, not commands IN. Dashboard handles intervention |
| Custom monitoring dashboard (from scratch) | Langfuse covers agent observability. Add a thin task status page later |

---

## Requirements (MoSCoW)

### Must Have — Phase 1 (Prove It Works)

| ID | Requirement |
|----|-------------|
| M1 | Nanoclaw running on Linux box (Docker, Node.js 22, Tailscale) |
| M2 | Hono HTTP API accepting task submissions from Nessie over Tailscale |
| M3 | Plainjob SQLite queue managing task lifecycle (queued → running → completed/failed) |
| M4 | Containerized agent execution via existing container-runner |
| M5 | Result reporting back to Nessie via HTTP callback (webhook) |
| M6 | Discord notifications: task started, completed, failed, errors |
| M7 | Basic health endpoint (`GET /health`) |
| M8 | Graceful shutdown (finish active containers on SIGTERM) |
| M9 | **10 proven task types** that work at >80% success rate |
| M10 | Langfuse integration for agent trace collection |

### Must Have — Phase 1b (Self-Improvement)

| ID | Requirement |
|----|-------------|
| M11 | Automatic post-mortem on every failure: what, why, root cause, prevention rule |
| M12 | Post-mortems written to `groups/{group}/CLAUDE.md` lessons section |
| M13 | Weekly retrospective task (scheduled) that analyzes all failures and refines rules |
| M14 | Success rate tracking per task type in Langfuse |

### Should Have — Phase 2

| ID | Requirement |
|----|-------------|
| S1 | Dashboard on Nessie: task status, queue depth, success rates, cost tracking |
| S2 | Scheduled autonomous tasks (leverage existing task-scheduler) |
| S3 | Budget tracking + daily cost alerts via Discord |
| S4 | Subagent coordination (planner agent decomposes complex tasks) |
| S5 | Supabase for shared state across future Nanoclaws |
| S6 | Neo4j knowledge graph: tasks ↔ lessons ↔ code modules ↔ agents |

### Could Have — Phase 3

| ID | Requirement |
|----|-------------|
| C1 | Redis upgrade for multi-Nanoclaw pub/sub coordination |
| C2 | Bidirectional dispatch (Alpha requests help from Nessie) |
| C3 | Approval workflows (certain actions need Dan's OK via dashboard) |
| C4 | Specialized skill packs per task type |

---

## Technical Design

### HTTP Task API (Hono)

Alpha exposes a simple HTTP API on port 3000 (Tailscale-only):

```
POST /tasks          — Submit a new task
GET  /tasks          — List all tasks (with status filter)
GET  /tasks/:id      — Get task details + result
DELETE /tasks/:id    — Cancel a task
GET  /health         — Health check
POST /tasks/:id/retry — Retry a failed task
```

**Task submission format:**
```json
{
  "taskId": "task-20260303-abc123",
  "type": "software_engineering",
  "priority": "normal",
  "instruction": "Write unit tests for the auth module. Target 80% coverage. Create a PR.",
  "context": {
    "repository": "https://github.com/dwidmaier/project",
    "branch": "main",
    "relevantFiles": ["backend/auth/"],
    "priorContext": "Uses JWT with refresh rotation."
  },
  "constraints": {
    "maxDurationMinutes": 30,
    "maxCostUsd": 5.00
  },
  "callbackUrl": "http://100.x.x.1:8080/callbacks/alpha"
}
```

**Result callback (Alpha → Nessie):**
```json
POST {callbackUrl}
{
  "taskId": "task-20260303-abc123",
  "status": "completed",
  "summary": "Created 12 tests. Coverage 34%→82%. PR #47.",
  "artifacts": { "prUrl": "...", "branch": "...", "filesChanged": [...] },
  "metrics": {
    "durationMs": 1097000,
    "estimatedCostUsd": 2.34
  },
  "lessonsLearned": ["Always check for existing test files first"]
}
```

This is simpler than Redis Streams, debuggable with `curl`, and Nessie can implement the callback endpoint easily.

### Job Queue (Plainjob + SQLite)

Plainjob sits between the HTTP API and the container runner:

```
HTTP POST /tasks → Validate (Zod) → Plainjob.enqueue(task, {priority}) → Worker picks up
  → ContainerRunner.spawn() → Agent executes → Result
  → POST callbackUrl + Discord notification + Langfuse trace
```

Plainjob handles:
- Persistent queue (survives restart)
- Priority ordering
- Retry with configurable backoff
- Concurrency limiting (matches MAX_CONCURRENT_CONTAINERS)

No Redis, no external dependencies. Just SQLite.

### Discord (Notification Channel to Dan)

Discord is outbound-only from agents to Dan. One bot per agent instance.

**Channel structure (minimal):**
```
Discord Server: "Agent Org"
├── #alpha-log        (all task events: started, completed, failed)
├── #nessie-log       (Nessie's status and actions)
├── #alerts           (errors requiring attention, budget warnings)
└── #daily-summary    (automated daily digest: tasks, costs, success rate)
```

**Example notifications:**

Task completed:
```
✓ task-abc123 completed (18m, $2.34)
"Created 12 unit tests for auth module. Coverage: 34% → 82%. PR #47."
```

Task failed:
```
✗ task-abc123 FAILED (attempt 3/3)
"Module '@auth/core' not found"
→ Post-mortem written. Lesson: verify dependencies before running tests.
```

Daily summary (scheduled, 6 PM):
```
Alpha Daily Summary — Mar 3, 2026
Tasks: 12 completed, 2 failed (86% success)
Cost: $28.40 | Avg: $2.37/task
Top failure: dependency resolution (2 occurrences)
New lessons learned: 1
```

### Langfuse (Agent Observability)

Self-hosted on Nessie (Docker Compose, ~256MB RAM). Receives traces from Alpha.

**What it tracks:**
- Every agent execution: prompt, tools used, tokens consumed, duration
- Success/failure per task type
- Failure clustering (identify common failure patterns)
- Cost per task, per day, per task type
- Latency distributions

**Integration point:** Instrument the agent-runner's `query()` call to push traces to Langfuse's HTTP API. ~20 lines of code.

**This replaces**: custom metrics in Redis hashes, custom dashboard for agent performance, manual failure analysis.

### Self-Improvement Loop (Phase 1b)

After every task completion (success or failure):

1. **Evaluate**: A lightweight Claude call evaluates the task outcome:
   - Did it achieve the goal?
   - What went well?
   - What went wrong?
   - What pattern does this failure match?

2. **Record**: Write evaluation to `groups/{group}/CLAUDE.md`:
   ```markdown
   ## Lessons Learned
   - [2026-03-03] Always run `npm install` before `npm test` on freshly cloned repos
   - [2026-03-03] Check for `.nvmrc` and use correct Node version before building
   ```

3. **Aggregate**: Weekly scheduled task runs a retrospective:
   - Pull all lessons from the week
   - Identify patterns across failures
   - Generate/refine rules
   - Push aggregated lessons to Langfuse for trend analysis

4. **Feedback**: Langfuse success rate per task type becomes the scoreboard. If a task type drops below 70% success, alert via Discord.

### Health & Monitoring

```
GET /health → {
  "status": "healthy|degraded|unhealthy",
  "uptime": 15782,
  "docker": "available",
  "activeContainers": 2,
  "queuedTasks": 1,
  "successRate24h": 0.85,
  "estimatedCostToday": 28.40
}
```

Nessie polls this endpoint every 30s. If unreachable for 90s, alert via Discord #alerts.

---

## 10 Proven Task Types (M9)

Focus Phase 1 on these — get them to >80% success before expanding:

| # | Task Type | Complexity | Expected Reliability |
|---|-----------|-----------|---------------------|
| 1 | Run test suite, report failures | Low | 95%+ |
| 2 | Lint/format codebase, create PR | Low | 95%+ |
| 3 | Update dependencies, check for breakage | Low | 90%+ |
| 4 | Write unit tests for a specified module | Medium | 80%+ |
| 5 | Fix a specific, well-described bug | Medium | 75%+ |
| 6 | Add TypeScript types to untyped module | Medium | 80%+ |
| 7 | Generate API documentation from code | Medium | 85%+ |
| 8 | Code review (analyze PR, post comments) | Medium | 80%+ |
| 9 | Scaffold a new module from a spec | Medium | 75%+ |
| 10 | Security audit (OWASP top 10 check) | Medium | 70%+ |

Each task type gets a dedicated system prompt snippet in CLAUDE.md with best practices, common pitfalls, and step-by-step procedures.

---

## New Files to Create

| File | Purpose | Est. Lines |
|------|---------|-----------|
| `src/api.ts` | Hono HTTP API (task CRUD + health endpoint) | ~150 |
| `src/job-queue.ts` | Plainjob wrapper: enqueue, worker, retry config | ~100 |
| `src/discord-notifier.ts` | Discord webhook/bot notifications (outbound only) | ~80 |
| `src/langfuse-client.ts` | Langfuse trace integration for agent runs | ~60 |
| `src/self-eval.ts` | Post-task evaluation + lesson extraction | ~100 |
| `src/health.ts` | Health endpoint + startup self-check | ~60 |

**Total new code: ~550 lines**

## Files to Modify

| File | Change |
|------|--------|
| `src/index.ts` | Add Hono server startup + job queue worker in `main()` |
| `src/config.ts` | Add API port, Langfuse URL, callback URL config |
| `package.json` | Add `hono`, `plainjob`, Langfuse SDK deps |
| `.env.example` | Add new config vars |
| `container/agent-runner/src/index.ts` | Add Langfuse trace instrumentation to `query()` call |

## Existing Files Unchanged

Everything else in nanoclaw stays untouched: container-runner, group-queue, IPC, db, task-scheduler, channels, router, types, mount-security, agent-runner (except Langfuse trace).

---

## Deployment Plan

### On the Linux Box (Alpha)

1. Install Docker + Node.js 22 (Tailscale already set up)
2. Clone fork: `git clone https://github.com/dwidmaier/nanoclaw.git`
3. `npm install` (includes existing deps)
4. `npm install hono plainjob @langfuse/langfuse` (new deps)
5. Apply Discord skill for Discord channel support
6. Configure `.env` (API key, Discord webhook URL, Langfuse URL, Nessie callback URL)
7. Build container image: `cd container && ./build.sh`
8. `npm run build && npm test`
9. Create systemd service (`nanoclaw-alpha.service`)
10. Verify: `curl http://100.x.x.2:3000/health`

### On Nessie's Mac Mini

1. Deploy Langfuse (Docker Compose): `docker compose up -d`
2. Install Redis (for Langfuse, not for Nanoclaw): Langfuse uses Postgres, actually — just its Docker Compose
3. Configure Nessie/OpenClaw to dispatch tasks via HTTP POST to `http://100.x.x.2:3000/tasks`
4. Set up callback endpoint on Nessie to receive results
5. (Phase 2) Dashboard web UI served from Nessie

### Discord Setup

1. Create Discord bot in Developer Portal
2. Create server "Agent Org" with channels: #alpha-log, #nessie-log, #alerts, #daily-summary
3. Generate webhook URLs for each channel
4. Add webhook URLs to Alpha's `.env`
5. (Simpler than a full Discord.js bot — webhooks are stateless, no connection to maintain)

---

## Success Criteria

| Criterion | Test |
|-----------|------|
| Task round-trip | `curl POST /tasks` → container executes → callback received on Nessie |
| Discord alerts | Every task completion/failure produces a Discord notification within 5s |
| Error recovery | Kill container mid-task → Plainjob retries → success or failure reported |
| Health monitoring | `GET /health` returns accurate status. Nessie detects Alpha down within 90s |
| Agent quality | 10 task types demonstrated with >80% success rate over 20+ attempts each |
| Self-improvement | Failed tasks produce post-mortems. Success rate for repeated task types improves over 1 week |
| Langfuse traces | Agent executions visible in Langfuse UI with token counts, duration, success/failure |
| 24-hour soak | 10+ tasks over 24h, no memory leaks, no orphan containers, stable health |

---

## Cost Estimate

- **API costs**: ~$2-5/task × 10-20 tasks/day = $20-100/day ($600-3,000/month)
- **Infrastructure**: $0 (spare hardware, self-hosted everything)
- **Break-even**: Each task needs to save ~15-30 min of Dan's time
- **Expected ROI**: Simple tasks (tests, lint, deps, docs) save 1-2 hours/day within month 1

---

## Risks

| Risk | Mitigation |
|------|-----------|
| Agent quality <80% on target tasks | Self-improvement loop + task-specific CLAUDE.md prompts. Narrow task scope until quality proven. |
| Langfuse adds complexity | It's Docker Compose on Nessie. If it breaks, Alpha still works — traces are optional. |
| Hono/Plainjob are less mature than Express/BullMQ | Both are actively maintained, well-tested. Risk is low for our throughput level (<100 tasks/day). |
| Nessie can't easily dispatch HTTP tasks | OpenClaw likely supports shell commands → `curl` works as a bridge. |
| Cost exceeds value | Start with $50/day budget cap. Track ROI per task type. Kill low-value task types. |

---

## Roadmap Summary

**Phase 1 (Week 1-2)**: Linux box running, HTTP API accepting tasks, containers executing, Discord notifications, Langfuse traces, health endpoint.

**Phase 1b (Week 2-3)**: Self-improvement loop (post-mortems, lessons, weekly retrospective). Prove 10 task types at >80% success.

**Phase 2 (Week 4-6)**: Dashboard on Nessie, scheduled autonomous tasks, budget tracking, Supabase shared state.

**Phase 3 (Month 2-3)**: Multi-Nanoclaw coordination (add Redis), Neo4j knowledge graph, approval workflows, bidirectional dispatch.
