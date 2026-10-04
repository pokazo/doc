# Landing + Auth Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Публичный лендинг pokazo.ru и вход клиентов (Яндекс ID + magic link), без генерации слайдов в этом плане.

**Architecture:** Монорепо: `apps/web` (Vite React) ходит на `apps/api` (Hono + Better Auth + Postgres). Сессия — httpOnly cookie. Дизайн только по `docs/design-system.md`.

**Tech Stack:** pnpm, Vite 6, React 19, Hono, Better Auth, Postgres, Unbounded + Golos Text.

**Specs:** `docs/design-system.md`, `docs/landing.md`, `docs/auth.md`.

## Global Constraints

- UI copy Russian; H1 exactly «Нейросеть для презентаций».
- Colors only `--pk-*` from design-system; do not use Inter or violet AI gradients.
- Yandex OAuth button uses official Yandex ID styling, not `--pk-spot`.
- `next` redirect must match `^/[\w\-/?=&]*$` or become `/`.
- No Slidev, no LLM, no BullMQ in this plan.
- Do not commit unless the user asks.

---

### Task 1: Monorepo skeleton

**Files:**
- Create: `package.json`, `pnpm-workspace.yaml`, `apps/web/package.json`, `apps/api/package.json`, `apps/web/vite.config.ts`, `apps/api/src/index.ts`
- Test: `apps/api/src/health.test.ts`

**Interfaces:**
- Consumes: nothing
- Produces: `GET /api/health` → `{ ok: true }`; web dev on `:5173`, api on `:3000`

- [ ] **Step 1:** Workspace `package.json` with `"packageManager": "pnpm@9"` and workspaces `apps/*`.
- [ ] **Step 2:** Hono app:

```ts
import { Hono } from "hono";
export const app = new Hono();
app.get("/api/health", (c) => c.json({ ok: true }));
```

- [ ] **Step 3:** Test with `app.request("/api/health")`, expect `ok: true`.
- [ ] **Step 4:** Vite React app, proxy `/api` → `http://localhost:3000`.

---

### Task 2: Design tokens

**Files:**
- Create: `apps/web/src/styles/tokens.css`, `apps/web/src/styles/global.css`
- Modify: `apps/web/src/main.tsx` import global.css

**Interfaces:**
- Produces: CSS variables `--pk-paper`, `--pk-ink`, `--pk-spot`, `--pk-stage`, fonts Unbounded + Golos Text

- [ ] **Step 1:** Copy hex values from `docs/design-system.md` into `:root` in `tokens.css`.
- [ ] **Step 2:** `body { background: var(--pk-paper); color: var(--pk-ink); font-family: "Golos Text", system-ui, sans-serif; }`
- [ ] **Step 3:** Screenshot `/` empty shell — paper background, no default Vite logo.

---

### Task 3: Landing UI

**Files:**
- Create: `apps/web/src/components/PkHeader.tsx`, `PkButton.tsx`, `PkInput.tsx`, `PkSlideFrame.tsx`, `apps/web/src/pages/Landing.tsx`
- Modify: `apps/web/src/App.tsx` route `/`

**Interfaces:**
- Consumes: tokens
- Produces: form submit → `navigate(/login?next=/app&topic=${encodeURIComponent(topic)})`

Copy from `docs/landing.md`: H1, subtitle, placeholder, CTA «Создать», header «Войти».

- [ ] **Step 1:** Header + hero + three steps + audience chips + footer stubs.
- [ ] **Step 2:** Empty topic: CTA disabled. Non-empty: navigates with `topic` query.
- [ ] **Step 3:** Viewport 390px and 1280px: CTA visible without horizontal scroll.

---

### Task 4: Auth module

**Files:**
- Create: `apps/api/src/auth.ts`, `apps/api/src/db.ts`, `apps/web/src/pages/Login.tsx`
- Modify: `apps/api/src/index.ts` mount Better Auth handler

**Interfaces:**
- Produces:
  - `GET /api/auth/me` → 401 or `{ id, email, name }`
  - Yandex sign-in + callback (Better Auth)
  - `POST /api/auth/sign-in/email` `{ email, next }`
  - `POST /api/auth/sign-out`

Env: `YANDEX_CLIENT_ID`, `YANDEX_CLIENT_SECRET`, `BETTER_AUTH_SECRET`, `DATABASE_URL`, `APP_ORIGIN`.

- [ ] **Step 1:** Postgres tables via Better Auth migrations (`user`, `session`, `account`, `verification`).
- [ ] **Step 2:** Register Yandex app redirect `http://localhost:3000/api/auth/callback/yandex`.
- [ ] **Step 3:** Login page: Yandex button + email field; checkbox ПДн required before submit.
- [ ] **Step 4:** Test: unauthenticated `/api/auth/me` is 401; after mocked session, 200 with email.
- [ ] **Step 5:** Reject `next=https://evil.com` → redirect `/`.

---

### Task 5: Gate `/app`

**Files:**
- Create: `apps/web/src/pages/AppHome.tsx` (form only, «генерация скоро»)
- Modify: router

**Interfaces:**
- Consumes: `GET /api/auth/me`
- Produces: if 401 → `/login?next=/app`; if 200 → show topic from query, no LLM call

- [ ] **Step 1:** Fetch me on load; redirect if 401.
- [ ] **Step 2:** Header shows name/email and «Выйти».
- [ ] **Step 3:** Manual check: land → type topic → Yandex → `/app` with topic still in the field.

---

## Done when

- `/` matches copy and tokens in docs.
- Yandex and magic-link create a session cookie.
- `/app` is unreachable logged out.
- No presentation generation yet.

## Execution Handoff

Plan complete and saved to `docs/superpowers/plans/2026-10-04-landing-auth.md`. Two execution options:

**1. Subagent-Driven (recommended)** — fresh subagent per task  
**2. Inline Execution** — this session, checkpoints

Which approach?
