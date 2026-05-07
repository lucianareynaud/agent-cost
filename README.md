# agent-cost

**FinOps cost attribution for AI code agents.**

Code agents (Kiro, Cursor, Claude Code, Copilot) charge per-request, per-token, or per-credit — but report cost at the session level, not the commit level. This makes it impossible to answer the basic FinOps question: *what am I paying per unit of useful output?*

`agent-cost` maps agent spend to git commits, producing a per-task cost ledger that reveals waste patterns, agent comparison data, and ROI signals at the granularity that matters.

## The problem

AI code agents have a **cost floor per invocation** driven by context window size, not task complexity. A 2-line type annotation fix and a 200-line feature implementation may cost similar amounts if the agent loads the same context. Without per-commit attribution, you can't distinguish high-value agent usage from expensive busywork.

Typical waste pattern (real data):

```
1.26 credits  →  fix: remove duplicate type annotation  (+1/-1 lines)
3.40 credits  →  feat: add routing functions             (+5/-0 lines)
0.80 credits  →  config: add setup.cfg                   (+1/-0 lines)
```

The fix and config tasks cost 2.06 credits combined for work that takes 30 seconds manually. That's 32% of total spend on zero-leverage tasks.

## Install

```bash
# Clone and symlink
git clone https://github.com/<you>/agent-cost.git
ln -s "$(pwd)/agent-cost/agent-cost" /usr/local/bin/agent-cost

# Or just copy
cp agent-cost /usr/local/bin/ && chmod +x /usr/local/bin/agent-cost
```

## Quick start

After any code agent session, in the same git repo:

```bash
# Kiro — credits shown in IDE after each task
agent-cost log 1.26 fix -a kiro -e 36

# Cursor — read credits from Settings > Usage or dashboard
agent-cost log 0.32 feat -a cursor -u usd -e 120

# Claude Code — use /cost output or ccusage session data
agent-cost log 0.55 feat -a claude-code -u usd -e 240
```

The tool auto-extracts from `git log`: commit hash, message, files changed, insertions, deletions.

### Task types

| Code | When to use | Waste signal |
|------|-------------|--------------|
| `fix` | Bug fix, lint fix, type error | **High** — often manual-editable |
| `feat` | New feature, multi-file | Low — expected high cost |
| `refac` | Refactor, rename, restructure | Medium |
| `test` | Test creation or modification | Medium |
| `doc` | Documentation, comments | **High** — low complexity |
| `config` | CI, deps, config files | **High** — low complexity |
| `other` | Uncategorized | — |

## Agent observability matrix

Each code agent exposes cost data differently. The table below maps what's actually available for FinOps tracking — based on official documentation, not assumptions.

| Dimension | Kiro | Cursor | Claude Code |
|-----------|------|--------|-------------|
| **Cost unit** | Credits (fractional, min 0.01) | USD credit pool (token-based) | USD or tokens |
| **Per-task visibility** | ✅ Shown after each task in IDE | ❌ Aggregate only (dashboard) | ✅ `/cost` per session (API), `/stats` (subscription) |
| **Real-time tracking** | ✅ Updated every 5 min in IDE dashboard | ⚠️ Dashboard at cursor.com, not real-time per-request | ✅ `/cost` command in terminal |
| **Token breakdown** | ❌ Credits are opaque unit | ⚠️ Token counts on dashboard, but not per-interaction | ✅ Input, output, cache read, cache creation tokens |
| **Model attribution** | ⚠️ Cost varies by model (Auto vs Sonnet 4 = 1.3x) but not labeled per task | ✅ Model selection shown per request | ✅ Model shown per interaction |
| **Programmatic access** | ❌ No API or export | ⚠️ Teams Analytics API (Enterprise only) | ✅ JSONL logs at `~/.claude/projects/`, parseable with `ccusage` |
| **Overage pricing** | $0.04/credit | Token-based at API rates | API: token rates; Subscription: included in plan |
| **Session data format** | None exportable | None exportable (individual plans) | JSONL with full token breakdown |

### Kiro (Amazon Q Developer)

**Cost model:** Credits — an opaque unit of work. Simple prompts < 1 credit; complex spec tasks > 1 credit. Credits are metered to 0.01 precision. Different models consume credits at different rates: Auto (default, cheapest) vs Sonnet 4 (1.3x) vs Opus (higher). All interactions consume credits: vibe mode, spec mode, spec refinement, task execution, agent hooks.

**What you can read:** After each task, the IDE shows "Credits used: X.XX" and elapsed time. The subscription dashboard (in-IDE) shows monthly credits used/remaining, updated every ~5 minutes.

**What you can't read:** No API, no export, no JSONL logs, no token breakdown. Credits are the only observable unit. You cannot reconstruct which model was used or how many tokens were consumed.

**FinOps extraction:**
```bash
# Manual: read credits + elapsed from IDE after each task
agent-cost log 1.26 fix -a kiro -e 36
```

### Cursor

**Cost model:** Hybrid subscription + token pool. Pro ($20/mo) includes a $20 credit pool consumed at model API rates. Auto mode is unlimited and free. Non-Auto requests (Claude Sonnet, GPT-4, etc.) deduct from the pool. Context size is the dominant cost driver — a single agentic prompt with 350k input tokens + 20k output tokens on Sonnet 4.5 can cost ~$1.35. Max mode adds a 20% surcharge.

**What you can read:** Web dashboard at `cursor.com/dashboard?tab=usage` shows token usage. Teams plans get per-request cost visibility and an Analytics API. Individual Pro plans show aggregate consumption only. A community extension and macOS widget (cursorusage.com) provide status-bar monitoring.

**What you can't read:** No per-commit or per-file attribution. No local log files. No programmatic API for individual plans. Per-request cost was briefly shown then removed from the individual dashboard.

**FinOps extraction:**
```bash
# Check dashboard credit balance before and after each task, compute delta
agent-cost log 0.32 feat -a cursor -u usd -e 120

# Alternative: use your own API key via OpenRouter for precise per-request cost
# (disables agent mode and some features)
agent-cost log 0.003 fix -a cursor -u usd -e 15
```

### Claude Code

**Cost model:** Fully transparent token-based pricing. API users pay per token at published rates (Sonnet 4.6: $3/$15 per MTok input/output). Subscription users (Pro $20, Max $100-$200/mo) get included token pools with 5-hour billing windows. Average: ~$6/dev/day on API, ~$100-200/dev/month.

**What you can read:** `/cost` shows per-session USD cost, API vs wall duration, token breakdown by type (input, output, cache read, cache creation), and lines changed. All session data persists as JSONL in `~/.claude/projects/*/sessions/*.jsonl`. The `ccusage` CLI parses these logs into daily, session, and billing-window reports. For API users, the Anthropic Console provides per-workspace cost tracking.

**What you can't read:** No native cross-session aggregation for subscription users (only API users get Console). Log rotation after ~7 days by default. No per-commit attribution built-in.

**FinOps extraction:**
```bash
# Option A: Read /cost output after each task
agent-cost log 0.55 feat -a claude-code -u usd -e 379

# Option B: Parse JSONL logs with ccusage
npx ccusage@latest session    # per-session cost breakdown
npx ccusage@latest daily      # daily aggregates
npx ccusage@latest blocks     # 5-hour billing window tracking

# Option C: Route through LLM gateway for production-grade observability
export ANTHROPIC_BASE_URL=http://localhost:8080/anthropic  # Bifrost or LiteLLM
```

Claude Code has the richest observability surface of the three. For production tracking, routing through an LLM gateway (Bifrost, LiteLLM) gives Prometheus metrics, per-request cost, and budget enforcement.

## Reports

```bash
agent-cost report                   # Full Pareto breakdown
agent-cost report --agent kiro      # Filter by agent
agent-cost summary --since 2025-03-01  # Period summary
agent-cost tail 20                  # Recent entries
agent-cost export                   # CSV path for pipeline integration
```

### Sample output

```
═══════════════════════════════════════════════════════════════
  CODE AGENT COST REPORT — 2026-03-19 — ALL AGENTS
═══════════════════════════════════════════════════════════════

  Total entries:     47
  Total cost:        38.92
  Total lines:       412 (+298 / -114)
  Avg cost/task:     0.828
  Avg cost/line:     0.094

───────────────────────────────────────────────────────────────
  BY AGENT
───────────────────────────────────────────────────────────────

  AGENT           COUNT     COST     UNIT   LINES  COST/LINE
  ──────────────  ──────  ────────  ──────  ─────  ──────────
  kiro                22    19.80  credits    198      0.100
  cursor              15     8.12      usd    134      0.061
  claude-code         10    11.00      usd     80      0.138

───────────────────────────────────────────────────────────────
  WASTE CANDIDATES (fix/doc/config with cost > 0.50)
───────────────────────────────────────────────────────────────

  1.26 credits | kiro         | fix    | +1/-1 | e19f4c3
  0.80 credits | kiro         | config | +1/-0 | d214a3b
```

## Pipeline integration

### Shell hook (post-commit reminder)

```bash
#!/usr/bin/env bash
# .git/hooks/post-commit
echo ""
echo "  Log agent cost: agent-cost log <cost> <type> -a <agent>"
echo ""
```

### Claude Code: semi-automated logging via ccusage

```bash
# After a Claude Code session, extract last session cost
LAST_COST=$(npx ccusage@latest session --json 2>/dev/null | \
  python3 -c "import sys,json; d=json.load(sys.stdin); print(f'{d[-1][\"total_cost\"]:.4f}')" 2>/dev/null)

if [ -n "$LAST_COST" ]; then
  agent-cost log "$LAST_COST" feat -a claude-code -u usd
fi
```

### CI cost badge (weekly cron)

```bash
CSV=$(agent-cost export)
WEEKLY_COST=$(awk -F',' -v since="$(date -u -v-7d +%Y-%m-%d)" \
  'NR>1 && $1 >= since { sum += $3 } END { printf "%.2f", sum }' "$CSV")
echo "Weekly agent cost: $WEEKLY_COST"
```

### Export to observability stack

The CSV at `~/.agent-cost/costs.csv` is designed for ingestion into any telemetry pipeline — OpenTelemetry filelog receiver, Prometheus Pushgateway, Grafana CSV plugin, or any BI tool.

### macOS shell aliases

```bash
# ~/.zshrc
alias ac="agent-cost"
alias acl="agent-cost log"
alias acr="agent-cost report"
```

## Complementary tools

| Tool | What it does | Works with |
|------|-------------|------------|
| [ccusage](https://github.com/ryoppippi/ccusage) | Parses Claude Code JSONL logs for daily/session/billing-window reports | Claude Code |
| [Claude-Code-Usage-Monitor](https://github.com/Maciek-roboblog/Claude-Code-Usage-Monitor) | Real-time TUI with burn rate predictions and session forecasting | Claude Code |
| [Cursor Usage Widget](https://cursorusage.com) | macOS menu bar widget monitoring Cursor dashboard | Cursor |
| [Bifrost](https://github.com/score-spec/bifrost) | Open-source LLM gateway with Prometheus metrics and budget enforcement | Claude Code (via ANTHROPIC_BASE_URL) |
| [LiteLLM](https://github.com/BerriAI/litellm) | Python proxy with per-key budget tracking across 100+ providers | Any agent with configurable base URL |

## Decision framework

**Use the agent when:**
- You don't know *what* to change (discovery required)
- The change spans multiple files needing coordination
- The task involves generation (new code, tests, docs from scratch)

**Skip the agent when:**
- You know the exact file, line, and fix
- The change is < 5 lines and mechanical
- Describing the task takes longer than doing it

## Architecture note

This tool operates at the **developer loop** level — manual attribution of agent cost to git commits. For **production pipeline** cost attribution (per-request, per-tenant, per-model), see architectures like [turnpike](https://github.com/lucianareynaud/turnpike), which instruments LLM gateways with OpenTelemetry for real-time cost, latency, and drift observability.

The two tools tell the same story at different scales: **cost without granular attribution is invisible cost.**

## License

MIT
