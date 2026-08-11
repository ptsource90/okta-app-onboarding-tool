# src/ — placeholder (planning phase)

This repository is currently in the **planning phase** (see root `README.md`
and `CLAUDE.md`). No application code has been written yet. `backend/` and
`frontend/` exist as empty placeholders so the intended project shape is
visible up front; each contains a `.gitkeep` only.

## Intended layout once implementation starts

```
src/
├── backend/
│   ├── OnboardingTool.Domain/
│   ├── OnboardingTool.Application/
│   ├── OnboardingTool.Infrastructure/
│   ├── OnboardingTool.Api/
│   └── OnboardingTool.sln
│   (+ a test project per layer, e.g. OnboardingTool.Domain.Tests)
└── frontend/
    ├── src/
    │   ├── api/
    │   ├── features/app-wizard/
    │   ├── components/
    │   ├── routes/
    │   └── auth/
    ├── index.html
    ├── package.json
    └── vite.config.ts
```

See `docs/02-architecture.md` for what belongs in each backend layer and
`docs/06-tech-stack.md` for the frontend library choices. When ready to
start implementation, ask the `software-engineer` subagent to scaffold the
backend solution and the Vite `react-ts` frontend app matching this layout.
