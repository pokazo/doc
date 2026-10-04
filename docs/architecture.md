# Архитектура Pokazo (MVP: генерация)

> Проверено: 2026-10-04 · источники: [Work with AI](https://sli.dev/guide/work-with-ai), [MCP](https://sli.dev/features/mcp), skill `skills/slidev/SKILL.md`, [#2475](https://github.com/slidevjs/slidev/issues/2475) · бренд: Pokazo / pokazo.ru

## Вердикт

MVP — **генератор**: тема → колода Slidev → веб-показ. Рендер и клики не пишем сами — берём **Slidev**. Логику «как писать слайды» не выдумываем — берём **готовые агенты/skills/MCP** как корпус промптов и инструменты правки `.md`. Редактор, аккаунт и «живой» PPTX — следующие этапы, не этот.

**С чего начать:** воркер = MCP-клиент к `slidev mcp` + официальный skill в промпте; потом `slidev build`.

Связанный документ: [functionality.md](./functionality.md). Спрос: [wordstat-top-queries.md](./wordstat-top-queries.md).

## Решение по стеку

| Слой | Выбор | Зачем |
|------|--------|--------|
| Модель колоды | `slides.md` (Slidev Markdown + YAML) | LLM пишет текст, не геометрию в пикселях |
| Рендер / показ | `@slidev/cli` (`slidev build` → статика) | Готовый плеер 16:9, темы, клики, Mermaid |
| Генерация | LLM + **официальный skill** + **официальный MCP** | Как в доке Slidev: знания отдельно, мутация колоды через tools |
| Инструменты агента | `slidev mcp slides.md` (stdio) | insert/update/list, не dump всего md |
| Референс продукта | [LSTM-Kirigaya/slidev-ai](https://github.com/LSTM-Kirigaya/slidev-ai) + [slidev-mcp](https://github.com/LSTM-Kirigaya/slidev-mcp) | Ближайшее готовое «приложение», не skill |
| API + воркер | **один Node/TS сервис** | Slidev — Node; второй язык (Kotlin/Python) всё равно shell-out в Node |
| HTTP | **Hono** | тонкий `POST /decks`, `GET /decks/:id` |
| Очередь | **BullMQ + Redis** | долгая генерация+сборка, ретраи, не держать HTTP |
| LLM SDK | **Vercel AI SDK** (`ai`) + **Zod** | outline `generateObject`; compose — tool-calling к MCP |
| LLM провайдер | OpenAI-compatible endpoint | ключ в env; позже YandexGPT/GigaChat тем же SDK |
| Сборка | `@slidev/cli` в Docker-job | spawn не на том же CPU, что HTTP, без сети из md |
| Логи | **pino** | job_id в каждой строке |
| Фронт MVP | Vite React `apps/web` | лендинг + login + /app; токены [design-system.md](./design-system.md) |
| Auth | Better Auth в `apps/api` | Яндекс ID + magic link; [auth.md](./auth.md) |

Не ядро платформы: Next.js, LangChain/LangGraph, Kotlin-воркер, Python, свой JSON-IR «как Gamma», Reveal.js.

## Как Slidev рекомендует строить агента

Канон: [Work with AI](https://sli.dev/guide/work-with-ai) + [MCP Server](https://sli.dev/features/mcp) (с v52.17). **Официального «нативного агента» нет** — открытый issue [slidevjs/slidev#2475](https://github.com/slidevjs/slidev/issues/2475) («Native Agent»). Рекомендация такая:

1. **Skill** (`npx skills add slidevjs/slidev`) — знания: синтаксис, layouts, клики, export. Не рантайм.
2. **MCP** — единственный API мутации колоды. Агент **не** должен переписывать весь `.md` текстом. Инструменты: `slidev-get-info`, `list/get/update/insert/remove/move-slide`, `goto-slide` (только с `slidev` dev). Stdio: `slidev mcp slides.md`. HTTP: `http://localhost:<port>/__mcp`.
3. **Агент** = любой MCP-клиент (Claude Code, Codex, Cursor, Copilot). VS Code: отдельный набор LM Tools (навигация, не полная мутация).
4. **Увидеть слайд:** в доке — `goto-slide` + браузер. antfu предлагает Playwright MCP. Скриншот в MCP ещё не upstream (#2475).

Для Pokazo это значит: воркер — **MCP-клиент к `slidev mcp`**, compose через `insert-slide` / `update-slide`, skill в system prompt. Свой `generateText` на весь файл — против рекомендации Slidev.

## Воркер: процесс и фреймворки

Cursor/Claude не в проде: тот же паттерн (skill + MCP), наш процесс — MCP-клиент.

```text
BullMQ job
  1. scaffold          пустой slides.md + headmatter
  2. slidev mcp        stdio на этот файл
  3. OutlineAgent      AI SDK generateObject(zod Outline)
  4. ComposeAgent      LLM + skill + tools MCP:
                       insert-slide / update-slide по outline
  5. MdGuard           whitelist layout, нет .vue / script
  6. SlidevBuilder     docker: npx slidev build
  7. Repair            build fail → MCP update-slide, не переписать файл целиком
  8. Persist           slides.md + dist/ + meta.json
```

| Берём | Не берём | Почему |
|-------|----------|--------|
| TypeScript / Node 22 | Kotlin API + Node worker на старте | два деплоя ради `slidev build` |
| Hono | Nest, Next Route Handlers | мало ручек, без SSR |
| BullMQ | In-process очередь, Temporal | Redis уже стандарт; Temporal рано |
| AI SDK + Zod | LangGraph, CrewAI, mastra | 2 вызова, не граф агентов |
| Zod outline | свободный JSON от модели | ломается число слайдов |
| Docker на build | `slidev build` в том же процессе API | CPU Vite + риск md |
| pnpm workspace | npm/monorepo-тулинг тяжелее | cli + worker + web |

Контракт outline (Zod, поля не раздувать): `title`, `slides[]` с `layout` из whitelist, `heading`, `bullets[]`. Compose **обязан** следовать этому JSON, не придумывать 20-й слайд.

Изоляция image: Node 22, `@slidev/cli`, тема(ы), без Playwright пока нет PDF. Сеть выключена. Таймаут сборки ~60s. Volume: `/work/:jobId`.

## Поток данных

```text
клиент                    pokazo-api                 worker
  │  POST /decks             │                          │
  │  {topic, locale, n}      │  enqueue                 │
  │◄──── job_id ─────────────┤                          │
  │                          │                          │
  │                          │        job               │
  │                          ├─────────────────────────►│
  │                          │                          │ 1. outline (LLM)
  │                          │                          │ 2. compose slides.md
  │                          │                          │    (skill + whitelist layout)
  │                          │                          │ 3. validate md
  │                          │                          │ 4. slidev build
  │                          │                          │ 5. optional: MCP patch overflow
  │                          │◄──── artifact ───────────┤
  │  GET /decks/:id          │    md + dist/ + meta     │
  │◄──── url просмотра ──────┤                          │
```

Артефакты джобы (хранить вместе):

| Файл | Роль |
|------|------|
| `input.json` | Тема, язык, число слайдов, аудитория |
| `outline.json` | Структура до Markdown |
| `slides.md` | Источник правды MVP |
| `dist/` | Статический плеер Slidev |
| `meta.json` | Модель, токены, статус, ошибки |

## Готовые агенты: что встраиваем

Пользователь **не** запускает Cursor. Агенты — библиотека для воркера.

| Компонент | Репо / дока | Как используем | ★ (на 2026-10-04) |
|-----------|-------------|----------------|-------------------|
| Официальный skill | [slidevjs/slidev](https://github.com/slidevjs/slidev) · [Work with AI](https://sli.dev/guide/work-with-ai) | Текст skill → system prompt compose-шага | движок ~49k |
| Официальный MCP | `slidev mcp slides.md` / HTTP `__mcp` | Пост-обработка: inspect, reorder, patch слайда | в CLI |
| slidev-ai | [LSTM-Kirigaya/slidev-ai](https://github.com/LSTM-Kirigaya/slidev-ai) | Смотреть UX и пайплайн, не форкать слепо в прод | 284 |
| slidev-mcp | [LSTM-Kirigaya/slidev-mcp](https://github.com/LSTM-Kirigaya/slidev-mcp) | Сверка инструментов MCP, если штатного CLI мало | 90 |
| slidev-skills (20) | [yoanbernabeu/slidev-skills](https://github.com/yoanbernabeu/slidev-skills) | Доп. правила layouts / code / export | 40 |
| SlideBlocks skill | [UniUni2000/slideblocks-skill](https://github.com/UniUni2000/slideblocks-skill) | Опционально: «улучшить колоду» вторым проходом | 71 |
| Visual loop | [camronh/slidev-agent](https://github.com/camronh/slidev-agent) | **Не MVP.** Позже: PNG → критика overflow | 1 |

Не в рантайм: VS Code Slidaiv, Copilot LM Tools, LangGraph-аддон [christian-bromann/slidev-agent](https://github.com/christian-bromann/slidev-agent) — это IDE, не мультиарендный SaaS.

## Ограничения рендера (жёстко)

| Правило | Почему |
|---------|--------|
| Whitelist layout: `default`, `cover`, `center`, `intro`, `two-cols`, `image-right`, `quote`, `section` | Свободный Vue/CSS от модели ломает сборку |
| Запрет кастомных `.vue` компонентов в MVP | Нет песочницы; skill иначе выдумывает теги |
| Тема одна на инсталляцию (или 2–3 пресета) | Gamma-вид за счёт темы Slidev, не за счёт ИИ-CSS |
| `slidev export --format pptx` = **картинки** на слайдах | Текст не редактируется в PowerPoint. Для рынка РФ — долг, не фича MVP |
| PDF через Playwright (`slidev export`) | Ок как «скачать», если нужен файл без PPTX |

## Границы системы MVP

**Внутри:** очередь генерации, хранение md+dist, публичная или signed ссылка на плеер, лимит слайдов, русский locale.

**Снаружи (не сейчас):** визуальный редактор, коллаб, бренд-кит клиента, нативный PPTX, биллинг (можно заглушка), мультипользовательские темы.

## Нефункциональное

| Тема | MVP |
|------|-----|
| Изоляция сборки | Docker/job: Node + `@slidev/cli`, таймаут, без сети из md |
| Секреты | ключ LLM только на воркере |
| Наблюдаемость | job_id, статус, сырой md при ошибке сборки |
| Язык UI | ru |

## Не брать / оговорки

- Конкуренты (Slidy) — React+Vite+свой редактор, не Slidev. Мы сознательно берём Slidev **только на этап генерации+показа**.
- PPTX «как у Slidy 98%» этим стеком **не** закрывается.
- Звёзды skills быстро устаревают; канон — официальные skill + MCP Slidev.

## История правок

| Дата | Что изменили |
|------|----------------|
| 2026-10-04 | Первичная запись: MVP генерация на Slidev + готовые агенты |
| 2026-10-04 | Стек воркера: Node 22, Hono, BullMQ, AI SDK+Zod, Docker `slidev build` |
| 2026-10-04 | Канон Slidev: skill + MCP tools; compose не generateText всего файла |
| 2026-10-04 | Лендинг Vite React; auth Better Auth + Яндекс ID |
