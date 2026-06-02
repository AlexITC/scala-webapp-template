# Wiringbits Web App Template — Developer Guide

## Quick links
| Topic | File |
|---|---|
| Running locally | [docs/guide/local-setup.md](docs/guide/local-setup.md) |
| Frontend (Scala.js / MUI) | [docs/guide/frontend.md](docs/guide/frontend.md) |
| Backend (Play / Tapir) | [docs/guide/backend.md](docs/guide/backend.md) |
| Database & infra | [docs/guide/data-infra.md](docs/guide/data-infra.md) |
| Testing | [docs/guide/testing.md](docs/guide/testing.md) |
| CI checklist | [docs/guide/ci-checklist.md](docs/guide/ci-checklist.md) |
| Anti-patterns | [docs/guide/antipatterns.md](docs/guide/antipatterns.md) |

## Architecture in one paragraph
Tapir endpoint definitions live in `lib/api/shared/` and are shared by both sides.
The backend (Play, `server/`) wires them to `serverLogic` in controllers, which
delegate to thin action objects, which call Anorm repositories.
The frontend (Scala.js + React + MUI 5, `web/`) reads the same models and calls the
API via an `ApiClient` accessed through `AppContext`.

## Layer map
```
lib/api/shared/     Tapir endpoints + request/response models (shared)
lib/common/shared/  Validated primitives: Email, Name, Password, …
server/             Play backend: controllers → actions → repositories
web/                Scala.js frontend: pages → widgets, AppContext, AsyncComponent
infra/              Ansible playbooks for deployment
```

## Self-check — run through this after every feature

- [ ] Did I add or update tests for this change?
- [ ] Do my tests follow conventions in [docs/guide/testing.md](docs/guide/testing.md)?
- [ ] If I wrote a component test, did I use `BrowserRouter` and jsdom, and is `null.asInstanceOf[API]` only used when the component never calls `ctx.api`?
- [ ] Do all my new `val`s at object level contain only plain JS literals — no `KeyAddingStage` or `ReactElement`?
- [ ] Are any React elements reused in the same render tree? If so, did I use `def` instead of `val`?
- [ ] Do my DB queries include `WHERE user_id = ?` for every user-owned table?
- [ ] Did I put new Tapir endpoint definitions in `lib/api/shared/`, not in `server/`?
- [ ] Did I run `sbt scalafmtAll` and re-stage any reformatted files before pushing?
