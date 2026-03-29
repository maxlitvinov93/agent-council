# Agent Council

Forked from [Carmack Council](https://github.com/SamJHudson01/Carmack-Council) — a multi-agent development framework for Claude Code.

Extended with **domain-specific experts** for trading systems, financial data products, and marketing.

## How It Works

Each project gets a `CLAUDE.md` that references experts from this library. When Claude Code reviews code, it loads the relevant expert's reference doc and applies their principles. No special skills needed — CLAUDE.md is automatically loaded.

```
Your Project/
├── CLAUDE.md          ← "when reviewing signals, read ernest-chan.md"
└── ...

~/agent-council/
└── references/        ← expert principle library (this repo)
```

## Expert Library

### Original Council (from Carmack Council)

| Expert | Domain | Reference Doc |
|--------|--------|--------------|
| Troy Hunt | Security | `security.md` |
| Martin Fowler | Refactoring / Structure | `refactoring.md` |
| Kent C. Dodds | Frontend Quality | `quality-frontend.md` |
| Matteo Collina | Backend Quality | `quality-backend.md` |
| Brandur Leach | Postgres Quality | `quality-postgres.md` |
| Simon Willison | LLM Pipeline Quality | `quality-llm.md` |
| Karri Saarinen | UI Quality | `quality-ui.md` |
| Vitaly Friedman | UX Quality | `quality-ux.md` |
| Kent Beck | Test Quality | `quality-testing.md` |

### Trading & Quantitative (NEW)

| Expert | Domain | Reference Doc |
|--------|--------|--------------|
| Ernest Chan | Signal quality, strategy evaluation, overfitting | `ernest-chan.md` |
| Nassim Taleb | Risk/ruin, position sizing, fat tails | `nassim-taleb.md` |

### Financial Data (NEW)

| Expert | Domain | Reference Doc |
|--------|--------|--------------|
| Edward Tufte | Data visualization, information design | `edward-tufte.md` |
| Aswath Damodaran | Valuation, financial metrics, scoring accuracy | `aswath-damodaran.md` |

### Marketing (NEW)

| Expert | Domain | Reference Doc |
|--------|--------|--------------|
| Rand Fishkin | SEO, LLM search optimization, organic growth | `rand-fishkin.md` |
| Sahil Bloom | Finance content creation, Twitter/X strategy | `sahil-bloom.md` |
| Pieter Levels | Solo founder growth, launches, communities | `pieter-levels.md` |

## Quick Start

1. Clone this repo: `git clone git@github.com:maxlitvinov93/agent-council.git ~/agent-council`
2. Copy the template: `cp ~/agent-council/templates/CLAUDE.md.template ./CLAUDE.md`
3. Fill in your project context and pick 2-4 relevant experts
4. Start working — Claude Code reads CLAUDE.md automatically

## Project Setup Examples

**Trading bot (Python):** Ernest Chan + Nassim Taleb
**Stock screener (FastAPI + Next.js):** Edward Tufte + Aswath Damodaran + Vitaly Friedman + Troy Hunt
**Marketing tasks:** Rand Fishkin + Sahil Bloom + Pieter Levels
**Web app (generic):** Kent C. Dodds + Matteo Collina + Troy Hunt + Martin Fowler

## Optimization vs Original Council

Original Carmack Council spawns **10 parallel subagents** per review (~500k-1M tokens).

Our approach: CLAUDE.md tells Claude **which 2-4 experts to apply** per task. Same quality, **5-7x fewer tokens**.

| Approach | Tokens per review |
|----------|------------------|
| Carmack Council (10 agents) | ~500k-1M |
| Agent Council (2-4 via CLAUDE.md) | ~80-200k |

## File Structure

```
references/              ← Expert principle libraries (20-25 principles each)
templates/
  CLAUDE.md.template     ← Template for new projects
skills/                  ← Original Carmack Council skills (optional)
dist/                    ← Built .skill packages (optional)
```

## Credits

- Original framework: [SamJHudson01/Carmack-Council](https://github.com/SamJHudson01/Carmack-Council)
- Expert personas inspired by real-world practitioners credited in each reference doc
