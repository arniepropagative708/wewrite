# Anti-AI Diagnostic Command Design

## Problem

Users (e.g., issue #2) configure `writing_persona: "midnight-friend"` but still fail AI detection. They have no way to know which anti-AI measures are actually in effect and which silently degraded. A one-command diagnostic tells them exactly what's working, what's missing, and what to fix first.

## Solution

Two entry points, one data flow:

1. **`scripts/diagnose.py`** — standalone Python script, programmatic checks, text/JSON output
2. **SKILL.md auxiliary function** — calls the script, adds LLM-powered cross-analysis

### Part 1: `scripts/diagnose.py`

**Location**: `scripts/diagnose.py` (same level as fetch_hotspots.py — it checks the whole skill, not just the toolkit)

**Invocation**:
```bash
python3 scripts/diagnose.py              # human-readable text
python3 scripts/diagnose.py --json       # structured JSON for agent consumption
```

**Path resolution**: The script resolves the skill root as its own parent directory (`Path(__file__).parent.parent`), same convention as other scripts.

**Check groups** (5 groups, each item yields `pass` / `warn` / `fail`):

#### Group 1: Dependencies
- Import each module from requirements.txt (`markdown`, `bs4`, `cssutils`, `requests`, `yaml`, `pygments`, `PIL`)
- Missing module = `fail` with install hint

#### Group 2: Config (`config.yaml`)
- File exists → `pass`; missing → `warn` (skip_publish + skip_image_gen)
- `wechat.appid` + `wechat.secret` present → `pass`; missing → `warn` (skip_publish)
- `image.api_key` present → `pass`; missing → `warn` (skip_image_gen)

#### Group 3: Style (`style.yaml`)
- File exists → check fields; missing → `fail`
- `writing_persona` field present → `pass`; missing → `warn` (defaults to midnight-friend)
- Corresponding persona file in `personas/` exists → `pass`; missing → `fail`

#### Group 4: Enhancement files
- `writing-config.yaml` exists → `pass`; missing → `warn` (using defaults, suggest optimize_loop.py)
- `playbook.md` exists → `pass`; missing → `warn` (no learned style, suggest "学习我的修改")
- `history.yaml` exists and has articles → `pass`; missing/empty → `warn` (no dedup, no dimension tracking)

#### Group 5: Dimension variance
- Read `history.yaml`, extract `dimensions` from last 3 articles
- All 3 have distinct dimension sets → `pass`; duplicates → `warn`
- Fewer than 3 articles → `skip` (not enough data)

**Anti-AI level scoring**:

Each check has a weight reflecting its impact on AI detection:

| Check | Weight | Rationale |
|-------|--------|-----------|
| style.yaml exists | 3 | No style = no persona, no tone control |
| writing_persona configured | 3 | Persona is the primary anti-AI lever |
| persona file exists | 2 | Without it, persona degrades to default |
| writing-config.yaml exists | 1 | Fine-tuning parameters, moderate impact |
| playbook.md exists | 2 | Learned style significantly improves human-ness |
| history.yaml has articles | 1 | Enables dimension dedup |
| dimension variance OK | 1 | Cross-article fingerprint diversity |
| config.yaml with wechat creds | 0 | Publish capability, no anti-AI impact |
| config.yaml with image key | 0 | Image gen, no anti-AI impact |
| Python dependencies | 0 | Prerequisite, not anti-AI specific |

Sum of weights for `pass` items / total possible (13) → percentage → level:
- 0-40% → `LOW`
- 41-75% → `MODERATE`
- 76-100% → `HIGH`

**Text output format**:
```
WeWrite Anti-AI Diagnostic
==========================

Dependencies
  [PASS] Python packages: all installed

Config
  [PASS] config.yaml: found
  [PASS] WeChat credentials: configured
  [WARN] Image API key: missing → image generation will be skipped

Style
  [PASS] style.yaml: found
  [PASS] writing_persona: midnight-friend
  [PASS] personas/midnight-friend.yaml: exists

Enhancement
  [WARN] writing-config.yaml: not found → using defaults (run optimize_loop.py to tune)
  [WARN] playbook.md: not found → no learned style (say "学习我的修改" after editing)
  [PASS] history.yaml: 12 articles

Dimension Variance
  [PASS] Last 3 articles have distinct dimensions

Summary: 7 passed, 3 warnings, 0 failures
Anti-AI level: ██████████░░ MODERATE (8/13)

Top recommendations:
  1. Run optimize_loop.py to generate writing-config.yaml
  2. Edit a generated article, then say "学习我的修改" to build playbook.md
```

**JSON output** (`--json`):
```json
{
  "checks": [
    {"group": "dependencies", "name": "python_packages", "status": "pass", "detail": "all installed"},
    {"group": "config", "name": "config_file", "status": "pass", "detail": "found"},
    {"group": "config", "name": "wechat_credentials", "status": "pass"},
    {"group": "config", "name": "image_api_key", "status": "warn", "detail": "missing", "impact": "skip_image_gen"},
    ...
  ],
  "summary": {
    "passed": 7,
    "warnings": 3,
    "failures": 0,
    "anti_ai_score": 8,
    "anti_ai_max": 13,
    "anti_ai_level": "MODERATE"
  },
  "recommendations": [
    "Run optimize_loop.py to generate writing-config.yaml",
    "Edit a generated article, then say \"学习我的修改\" to build playbook.md"
  ],
  "files": {
    "config_yaml": true,
    "style_yaml": true,
    "writing_config_yaml": false,
    "playbook_md": false,
    "history_yaml": true,
    "persona_file": "personas/midnight-friend.yaml"
  }
}
```

The `recommendations` list is ordered by impact (highest weight missing items first). The `files` map gives the agent quick access to which files exist without re-checking.

### Part 2: SKILL.md Auxiliary Function

**Trigger**: User says "诊断反 AI 配置" / "检查配置" / "为什么 AI 检测没过"

**Agent flow**:

1. Run `python3 {skill_dir}/scripts/diagnose.py --json`
2. If any `fail` items → report them, suggest fixes, stop here
3. If all `pass` or only `warn` → proceed to LLM deep analysis:
   - Read `style.yaml`: extract `tone`, `voice`, `writing_persona`
   - Read the active persona YAML file
   - Read `writing-config.yaml` (if exists)
   - Read `history.yaml` last 5 entries (if exists)
4. LLM cross-analysis checks:

| Check | What to look for | Example issue |
|-------|-----------------|---------------|
| tone ↔ persona consistency | tone/voice keywords vs persona's voice_density, emotional_arc, avoid list | tone="严谨客观" with midnight-friend (极度口语化) |
| writing-config danger params | Values that produce AI-like output | `emotional_arc: flat`, `paragraph_rhythm: structured`, `closing_style: summary` |
| history persona usage | Whether persona is actually being used in recent articles | history entries with no `writing_persona` field |
| WebSearch degradation | Recent articles' `topic_source` showing LLM fallback | All recent articles lack real material anchoring |

5. Output natural language report with prioritized action items

**What it does NOT do**:
- Does not run humanness_score.py (requires an existing article)
- Does not modify any config files (diagnose + recommend only)
- Does not re-run the full pipeline

### SKILL.md Changes

Add to the "辅助功能" section after existing entries:
```
- 用户说"诊断配置"/"检查反AI" → 运行 diagnose.py --json，结合 LLM 分析输出报告
```

Add to Step 8c "后续操作" table:
```
| 诊断配置 / 检查反AI | 运行 diagnose.py + LLM 交叉分析 |
```

## Files Changed

| File | Change |
|------|--------|
| `scripts/diagnose.py` | New file — diagnostic script |
| `SKILL.md` | Add auxiliary function entry + Step 8c row |
| `README.md` | Add diagnose command to "Toolkit 独立使用" section |
