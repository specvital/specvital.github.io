# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

Central documentation repository for the Specvital platform (https://specvital.github.io)

- VitePress-based static documentation site
- Multi-language support (English `/en/`, Korean `/ko/`)
- Auto-deployed to GitHub Pages on push to main branch

## Commands

```bash
# Dev server
pnpm docs:dev

# Build
pnpm docs:build

# Preview build
pnpm docs:preview

# Install dependencies
just deps

# Format all files
just lint all
```

## Architecture

```
docs/
├── .vitepress/config.mts    # VitePress config (navigation, sidebar)
├── en/                      # English docs
│   ├── prd/                 # Product Requirements Document
│   └── adr/                 # Architecture Decision Records
│       ├── core/            # Core microservice ADR
│       ├── collector/       # Collector microservice ADR
│       └── web/             # Web microservice ADR
└── ko/                      # Korean docs (mirrors en/ structure)
```

## Documentation Rules

### CRITICAL: Adding New Documents

> **MUST follow ALL steps. Skipping ANY step is a violation.**

1. Create in both `/en/` and `/ko/` with identical structure
2. **Update `docs/.vitepress/config.mts`** - sidebar navigation
3. **Update ALL relevant `index.md` files** in both EN and KO:

#### ADR Document Example (e.g., `collector/08-xxx.md`)

Must update these 4 files:

```
docs/en/adr/index.md           ← Parent ADR index (Collector Repository section)
docs/ko/adr/index.md           ← Parent ADR index (Collector 리포지토리 section)
docs/en/adr/collector/index.md ← Collector ADR index
docs/ko/adr/collector/index.md ← Collector ADR index
```

#### PRD Document Example (e.g., `prd/07-xxx.md`)

Must update these 2 files:

```
docs/en/prd/index.md
docs/ko/prd/index.md
```

> **DO NOT assume "collector/index.md updated = done". Parent index.md MUST also be updated.**

### Language Convention

- English docs (`/en/`): Write in English
- Korean docs (`/ko/`): Write in Korean
- Follow document language rules regardless of conversation language

### CRITICAL: Korean Writing Style

> **MUST use noun-ending style. Verb endings are prohibited.**

- Use noun-ending style (명사형 종결): "~생성", "~필요", "~제공", "~처리", "~확인"
- **NEVER use verb-ending style** (동사 종결어미): "~한다", "~이다", "~합니다", "~됩니다"
- Example:
  - ✅ "정적 분석을 통한 테스트 카운트"
  - ✅ "데이터 무결성 검증"
  - ✅ "API 호출 최소화"
  - ❌ "정적 분석을 통해 테스트를 카운트한다"
  - ❌ "데이터 무결성을 검증합니다"
  - ❌ "API 호출이 최소화됩니다"
