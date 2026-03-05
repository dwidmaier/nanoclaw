# Lessons Learned

Patterns and mistakes to avoid, updated after every correction.

## Session 2026-03-04 (PRD Design)

- Don't strip upstream channels — channels auto-skip when env vars are missing
- Discord is outbound notifications only, not a control plane
- Dashboard for oversight lives on Nessie, not in Discord
- Integrate production-ready components (Hono, Plainjob, Langfuse), don't reinvent
- Prove agent quality before building infrastructure
