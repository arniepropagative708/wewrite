# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

WeWrite is a WeChat public account (公众号) content generation AI skill. It automates the full workflow from trending topic discovery to draft box publishing. It works as both a Claude Code skill (via SKILL.md) and an OpenClaw-compatible skill (via `dist/openclaw/`).

The core pipeline is defined in SKILL.md (Steps 1-8): environment check → topic selection → framework + material collection → writing → SEO/anti-AI verification → visual AI → formatting/publishing → wrap-up.

## Commands

```bash
# Install dependencies
pip install -r requirements.txt

# Toolkit CLI
python3 toolkit/cli.py preview article.md --theme sspai       # Preview as HTML
python3 toolkit/cli.py publish article.md --cover cover.png --title "标题"  # Publish to WeChat
python3 toolkit/cli.py gallery                                  # Browse all 16 themes
python3 toolkit/cli.py themes                                   # List theme names
python3 toolkit/cli.py image-post img1.jpg img2.jpg -t "标题"   # Image post (carousel)

# Data collection scripts
python3 scripts/fetch_hotspots.py --limit 20                   # Trending topics
python3 scripts/seo_keywords.py --json "关键词1" "关键词2"      # SEO keyword analysis
python3 scripts/fetch_stats.py <article_id>                     # WeChat article stats
python3 scripts/humanness_score.py article.md --verbose         # AI detection scoring (11 checks)

# Build OpenClaw-compatible skill (also runs in CI on push to main)
python3 scripts/build_openclaw.py
```

No formal test suite exists. CI only rebuilds the OpenClaw version on push to main.

## Architecture

### Dual Nature: Skill + Toolkit

- **As a skill** (SKILL.md): An agent-orchestrated 8-step pipeline. The LLM reads SKILL.md and executes steps, calling Python scripts as tools. Reference docs in `references/` are loaded on-demand by the agent at specific steps.
- **As a standalone toolkit** (`toolkit/cli.py`): A Python CLI for Markdown→WeChat HTML conversion and publishing, usable independently of the skill.

### Key Directories

- `scripts/` — Data collection utilities (hotspots, SEO, stats) and build tools. Called by the agent during pipeline execution.
- `toolkit/` — Markdown→WeChat HTML converter, theme engine, WeChat API client, image generation. The CLI entry point is `toolkit/cli.py`.
- `personas/` — 5 YAML writing personality presets controlling tone, data presentation, emotional arc. Loaded in Step 4b.
- `references/` — Agent-loaded docs (writing rules, frameworks, SEO, topic scoring). These are NOT code — they are instruction sets the LLM reads and follows.
- `toolkit/themes/` — 16 YAML theme definitions. Parsed by `toolkit/theme.py`, applied as inline CSS by `toolkit/converter.py`.

### Formatting Pipeline (toolkit)

`converter.py` is the core: Markdown → HTML with inline styles + WeChat compatibility fixes (CJK spacing, bold punctuation, list→section conversion, external links→footnotes, dark mode attributes). WeChat strips `<style>` tags, so all CSS must be inlined. Themes are YAML files defining colors and base CSS; `theme.py` parses them, `converter.py` applies them.

### OpenClaw Compatibility

`scripts/build_openclaw.py` transforms SKILL.md for OpenClaw: replaces `{skill_dir}` with `{baseDir}`, renames tool references (WebSearch→web_search, etc.), copies referenced files. CI runs this on push to main and commits to `dist/openclaw/`.

### Configuration Files

- `config.yaml` (from `config.example.yaml`) — WeChat API credentials + image API key. Missing → graceful degradation (skip_publish, skip_image_gen).
- `style.yaml` (from `style.example.yaml`) — User's writing profile (name, topics, tone, persona, theme). Auto-created via onboard flow on first run.
- `writing-config.yaml` (from `writing-config.example.yaml`) — Writing parameters (sentence variance, idiom density, etc.). Optimized per-user via the "优化参数" auxiliary function in SKILL.md.

All three are .gitignored — each user generates their own.

### Graceful Degradation

The pipeline never hard-fails. Missing config → skip_publish/skip_image_gen flags. Script failures → WebSearch or LLM fallback. Image gen fails → output prompts only. These flags are set in Step 1 and automatically respected by later steps.

## Language & Conventions

- All code is Python 3.11+. No type checking or linter configured.
- Commit messages are in Chinese, format: `type: description` (e.g., `fix: ...`, `chore: ...`).
- The project language (README, SKILL.md, comments, references) is Chinese.
