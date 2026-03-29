# Agent Council

Библиотека экспертных reference docs для Claude Code. Каждый doc — 20-25 принципов от реального эксперта, заточенных под конкретный домен.

## Как это работает

```
Твой проект/
├── CLAUDE.md    ← ссылки на нужных экспертов

~/agent-council/
└── references/  ← библиотека принципов (этот репо)
```

1. В проекте создаёшь `CLAUDE.md` (из шаблона `templates/CLAUDE.md.template`)
2. Указываешь какие эксперты нужны и когда их вызывать
3. Claude Code автоматически читает CLAUDE.md при старте сессии
4. При ревью/написании кода — загружает reference doc нужного эксперта

**Не нужны субагенты, скиллы или специальные команды.** Просто CLAUDE.md + reference docs.

## Структура

```
references/
├── programming/           # Код и архитектура
│   ├── david-beazley.md       Python async, concurrency, event loop
│   ├── sebastian-ramirez.md   FastAPI, Pydantic, dependency injection
│   ├── kent-dodds-react.md    React, Next.js, React Query, components
│   ├── quality-frontend.md    Frontend (Kent C. Dodds, tRPC/CSS Modules стек)
│   ├── quality-backend.md     Backend/Node.js (Matteo Collina)
│   ├── quality-postgres.md    PostgreSQL (Brandur Leach)
│   ├── quality-testing.md     Testing (Kent Beck)
│   └── quality-llm.md         LLM pipelines (Simon Willison)
│
├── trading/               # Трейдинг и quant
│   ├── ernest-chan.md         Signal quality, overfitting, edge calculation
│   └── nassim-taleb.md        Risk/ruin, position sizing, fat tails
│
├── financial/             # Финансовые данные
│   ├── edward-tufte.md        Data visualization, charts, tables
│   └── aswath-damodaran.md    Valuation, scoring, financial metrics
│
├── marketing/             # Рост и маркетинг
│   ├── rand-fishkin.md        SEO, LLM search optimization
│   ├── sahil-bloom.md         Finance content, Twitter/X strategy
│   └── pieter-levels.md       Solo founder growth, launches, communities
│
├── universal/             # Любой проект
│   ├── security.md            Security (Troy Hunt)
│   ├── refactoring.md         Architecture (Martin Fowler)
│   ├── quality-ui.md          UI design (Karri Saarinen)
│   └── quality-ux.md          UX patterns (Vitaly Friedman)
│
└── spec-templates/        # Шаблоны спецификаций
    ├── feature-spec.md
    ├── product-spec.md
    ├── small-change.md
    ├── acceptance-criteria-guide.md
    ├── boundary-examples.md
    └── anti-patterns.md

templates/
└── CLAUDE.md.template     # Шаблон для нового проекта
```

## Быстрый старт

```bash
# 1. Клонируй (один раз)
git clone git@github.com:maxlitvinov93/agent-council.git ~/agent-council

# 2. В новом проекте — скопируй шаблон
cp ~/agent-council/templates/CLAUDE.md.template ./CLAUDE.md

# 3. Заполни: проект, стек, выбери 2-4 эксперта, список библиотек для Context7

# 4. Работай — Claude Code читает CLAUDE.md автоматически
```

## Подбор экспертов по типу проекта

| Тип проекта | Рекомендуемые эксперты |
|-------------|----------------------|
| **Trading bot (Python)** | David Beazley + Ernest Chan + Nassim Taleb |
| **Stock screener (FastAPI + Next.js)** | Sebastián Ramírez + Kent C. Dodds + Edward Tufte + Aswath Damodaran |
| **Web app (Node.js)** | Matteo Collina + Kent C. Dodds (original) + Troy Hunt + Martin Fowler |
| **Web app (Python)** | Sebastián Ramírez + Kent Beck + Troy Hunt |
| **LLM pipeline** | Simon Willison + Kent Beck + Troy Hunt |
| **Marketing tasks** | Rand Fishkin + Sahil Bloom + Pieter Levels |

## Формат reference doc

Каждый файл — единый формат:

```markdown
# [Title] — [Expert Name]

Philosophy: [Expert background]
Stack context: [Relevant stack]

## Principle 1: [Strong statement]

*[Expert]: "[Quote]"*

[Explanation]

### What to check
**[Sub-topic]**
- [Concrete check item]
- Severity: **P1/P2/P3**
```

**P1** = баг, уязвимость, потеря данных/денег — фиксить сейчас
**P2** = tech debt который будет стоить дорого потом — фиксить скоро
**P3** = clarity, polish, naming — по возможности

## Добавление нового эксперта

1. Создай файл в нужной категории (`references/[category]/[name].md`)
2. Следуй формату выше: 20-25 пронумерованных принципов
3. Каждый принцип: цитата + объяснение + "What to check" + severity
4. Добавь эксперта в CLAUDE.md проектов где он нужен

## Context7 MCP

Для актуальных доков библиотек используется [Context7](https://context7.com) MCP сервер. Установлен глобально, вызывается автоматически при написании кода (правило в CLAUDE.md каждого проекта).

## Credits

- Reference doc формат вдохновлён [Carmack Council](https://github.com/SamJHudson01/Carmack-Council)
- Эксперты: реальные специалисты, принципы из их публикаций и книг
