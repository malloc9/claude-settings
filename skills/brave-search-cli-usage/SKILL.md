---
name: brave-search-cli-usage
description: Use when needing to use Brave Search CLI (bx) for web searches, RAG grounding, AI answers, or research automation. Covers correct syntax, Goggles patterns, API key setup, and command selection.
---

# Brave Search CLI Usage

## Overview
Brave Search CLI (`bx`) is a zero-dependency CLI for Brave Search API, purpose-built for AI agents and LLMs. It replaces the search → scrape → extract pipeline with pre-extracted, token-budgeted content.

**Key insight:** Use `context` by default for technical documentation (RAG grounding). Use `answers` for synthesized explanations. Use `web` when you need `site:` operators or result filtering.

## When to Use

**Use bx when:**
- You need current information beyond training data
- Researching errors, APIs, or code patterns
- Gathering documentation for code generation
- Performing competitive analysis or security research
- Automating research workflows with token budgets

**Don't use bx when:**
- The information is in your training dataset (pre-2024)
- You need real-time data (use `news` with freshness filters)
- You're offline (bx requires network)

## Core Pattern: Correct Command Syntax

### ❌ Common Mistakes (from baseline tests)

```bash
# WRONG: --query flag doesn't exist
bx --query "search term"

# WRONG: --token-budget is not a valid flag
bx context "search" --token-budget 4096

# WRONG: positional arguments and flags mixed incorrectly
bx context --query "term" --max-tokens 4096  # --query invalid
```

### ✅ Correct Patterns

```bash
# Simple search (context is default)
bx "your search query"

# Context command with token budget (RAG grounding)
bx context "your search query" --max-tokens 4096

# AI answers (streaming)
bx answers "explain Rust lifetimes"

# AI answers (non-streaming, single JSON)
bx answers "question" --no-stream | jq .

# Web search with count
bx web "site:docs.rs axum middleware" --count 5

# News with freshness filter (pd = past day)
bx news "npm security advisory" --freshness pd
```

**FLAG NAMES MATTER:** Always use `--max-tokens` (not `--budget`, `--token-budget`, or `-t`).

## Quick Reference

| Command | Use Case | Output Shape | Typical Flags |
|---------|----------|--------------|---------------|
| `context` | **RAG/grounding** (default) | `.grounding.generic[].snippets[]` | `--max-tokens`, `--threshold`, `--goggles` |
| `answers` | AI-generated explanation | OpenAI-compatible streaming | `--no-stream`, `--enable-research`, `--model` |
| `web` | Full search with operators | `.web.results[]`, `.news[]` | `--count`, `--result-filter`, `--goggles` |
| `news` | News articles | `.results[]` | `--freshness`, `--count` |

**Exit codes:** 0=success, 1=client error, 2=usage error, 3=auth error, 4=rate limited, 5=server error

## Goggles: Custom Re-Ranking

**IMPORTANT DISTINCTION:**

| Method | Purpose | Syntax | Where to Use |
|--------|---------|--------|--------------|
| `site:` operators | Filter results in query | `site:docs.rs axum` | Inside query string (works with any command) |
| Goggles DSL | Re-rank/boost/discard | `$boost=5,site=docs.rs` | In `--goggles` flag (one rule per line) |
| `--include-site` | Easy domain allowlist | `--include-site docs.rs` | As flag (multiple allowed) |
| `--exclude-site` | Easy domain blocklist | `--exclude-site w3schools.com` | As flag (multiple allowed) |

**Do NOT mix them:** `--goggles "site:docs.rs"` is **WRONG**. Use `$boost=5,site=docs.rs` in Goggles.

### Inline Rules (Zero Setup) - Goggles DSL

```bash
# Boost official Rust sources, discard tutorial spam
bx context "serde custom deserializer" \
  --goggles '$boost=5,site=docs.rs
$boost=5,site=crates.io
$boost=3,site=github.com
$discard,site=geeksforgeeks.org
$discard,site=w3schools.com' --max-tokens 4096
```

**Goggles DSL Quick Reference:**

| Rule | Effect |
|------|--------|
| `$boost=N,site=DOMAIN` | Boost domain (N=1-10) |
| `$downrank=N,site=DOMAIN` | Demote domain (N=1-10) |
| `$discard,site=DOMAIN` | Remove domain entirely |
| `/path/$boost=N` | Boost matching URL paths |
| `*pattern*$boost=N` | Wildcard URL matching |
| Generic `$discard` | Allowlist mode (discard all unmatched) |

### Domain Shortcuts - Easiest for Simple Cases

```bash
# Include-only specific domains
bx "query" --include-site docs.rs --include-site github.com

# Exclude specific domains
bx "query" --exclude-site medium.com --exclude-site w3schools.com
```

**Use `--include-site`/`--exclude-site` when:** You just need to filter domains. Use `--goggles` for advanced ranking (boosting some sites more than others, path-based rules, wildcards).

### From a File (Reusable Goggles)

```bash
# Create reusable goggles file
cat > .goggle << 'EOF'
$boost=5,site=docs.rs
$boost=3,site=github.com
/blog/$downrank=3
$discard,site=geeksforgeeks.org
EOF

# Reuse across queries
bx context "tokio async" --goggles @.goggle --max-tokens 4096
bx context "serde serialize" --goggles @.goggle --max-tokens 4096
```

## Token Budgeting

Control output size with `--max-tokens` and related flags:

```bash
# Limit total tokens returned
bx context "query" --max-tokens 4096

# Limit per URL (prevents one page from dominating)
bx context "query" --max-tokens 4096 --max-tokens-per-url 1024

# Limit number of URLs (default varies)
bx context "query" --max-tokens 4096 --max-urls 10

# Combine all three for precise control
bx context "query" --max-tokens 4096 --max-tokens-per-url 1024 --max-urls 5
```

**Thresholds:** Use `--threshold strict` for only high-relevance results, or `--threshold moderate` (default) for broader coverage.

## API Key Configuration

**Three methods (priority order):**

1. **Config file** (recommended for Claude Code):
   ```bash
   bx config set-key YOUR_API_KEY
   # Stored at: ~/.config/brave-search/api_key (Linux)
   #            ~/Library/Application Support/brave-search/api_key (macOS)
   ```

2. **Environment variable**:
   ```bash
   export BRAVE_SEARCH_API_KEY=YOUR_API_KEY
   ```

3. **Flag** (least secure, visible in process list):
   ```bash
   bx --api-key KEY "query"
   ```

**Security:** Prefer config file or env var. Use `bx config set-key` without argument to enter key interactively (avoids shell history).

## Error Handling

bx returns distinct exit codes. Handle them appropriately:

```bash
# Check exit code and take action
bx context "query" --max-tokens 4096
case $? in
  0) echo "Success" ;;
  1) echo "Fix query/parameters" ;;
  2) echo "Fix CLI arguments" ;;
  3) echo "Check API key or plan: bx config show-key" ;;
  4) echo "Rate limited - retry after delay" ;;
  5) echo "Network/server error - retry with backoff" ;;
esac
```

**Rate limiting (429):** Implement exponential backoff. Consider upgrading plan for higher limits.

## Command Selection Guide

| Your Need | Command | Why |
|-----------|---------|-----|
| Look up docs, errors, code patterns | `context` | Pre-extracted text, token-budgeted, no scraping |
| Get synthesized explanation | `answers` | AI-generated with citations |
| Search specific site (`site:`) | `web` | Supports search operators |
| Find discussions/forums | `web --result-filter discussions` | Forums often have solutions |
| Check latest versions/releases | `context` or `news --freshness pd` | Fresh beyond training data |
| Research CVEs/security | `context` or `news` | CVE details, advisories |
| Filter to official sources | `--include-site` or `--goggles` | Boost docs.rs, crates.io, github.com |

## Implementation Checklist

**Before running bx:**
- [ ] API key configured? (`bx config show-key` to verify)
- [ ] bx installed? (`which bx` or `bx --version`)
- [ ] Chosen right command? (`context` for docs, `answers` for synthesis)
- [ ] Set token budget? (prevent excessive output with `--max-tokens`)
- [ ] Need domain filtering? (use `--include-site`, `--exclude-site`, or `--goggles`)

**Constructing queries:**
- [ ] Query is specific enough (include error messages, crate names, version numbers)
- [ ] Use `site:` operators in `web` if targeting specific domains
- [ ] Avoid tutorial spam in results (apply Goggles proactively)

## Common Mistakes

| Mistake | Why It's Wrong | Fix |
|---------|----------------|-----|
| `bx --query "term"` | `--query` flag doesn't exist | Use positional arg: `bx "term"` |
| `bx context --token-budget 4096` | Invalid flag name | Use `--max-tokens 4096` |
| `bx context --budget 4096` | Invalid flag name | Use `--max-tokens 4096` |
| Using `web` for documentation | Doesn't pre-extract content | Use `context` instead |
| `--goggles "site:docs.rs"` | `site:` is query operator, not Goggles DSL | Use `$boost=5,site=docs.rs` in Goggles |
| Using `site:` in Goggles | Confuses query syntax with Goggles DSL | `site:` goes in query string, not `--goggles` |
| Forgetting `--max-tokens` | Unbounded output, token waste | Always set token budget |
| Not checking exit codes | Silent failures ignored | Check `$?` after bx commands |
| Using `--api-key` in scripts | Key visible in process list | Use config file or env var |

## Real-World Workflows

### Debugging an Error

```bash
# 1. Broad RAG search with token budget
bx context "TypeError cannot unpack non-iterable NoneType" --max-tokens 4096

# 2. Too general? Narrow with site restriction
bx web "site:stackoverflow.com TypeError cannot unpack" --count 5

# 3. Need synthesis? Ask for AI answer
bx answers "why does unpacking None cause TypeError in Python" --no-stream | jq .
```

### Evaluating a Rust Crate

```bash
# Search for security issues and maintenance status
bx context "reqwest crate security vulnerabilities maintenance 2026" \
  --goggles '$boost=5,site=docs.rs
$boost=5,site=crates.io
$boost=3,site=github.com
$discard,site=geeksforgeeks.org' \
  --max-tokens 4096 \
  --threshold strict

# Check recent news for advisories
bx news "reqwest security advisory" --freshness pd
```

### Pre-Upgrade Research

```bash
# Check breaking changes in next version
bx context "Next.js 15 breaking changes migration guide" --max-tokens 8192

# Verify recent activity
bx news "Next.js 15 release" --freshness pm
```

### Iterative RAG Loop

```bash
# Broad initial search
bx "axum middleware authentication" --max-tokens 4096

# If too general, narrow scope
bx "axum middleware tower layer authentication example" \
  --threshold strict --max-tokens 4096

# Still unclear? Request synthesis
bx answers "how to implement JWT auth middleware in axum" --enable-research
```

---

## Red Flags - STOP and Fix

- Using `--query` or `--token-budget` flags → These don't exist
- Forgetting `--max-tokens` → Token waste, expensive
- Using `web` for docs → Use `context` instead
- Not checking `bx` exists → `which bx || echo "install required"`
- Ignoring exit codes → Silent failures
- Passing API key via `--api-key` in shared environment → Use config/ env var

## Rationalization Table

| Excuse | Reality |
|--------|---------|
| "I can guess the syntax" | Flags are specific; guessing causes errors (Test #1) |
| "Other search tools use similar flags" | bx uses different conventions (positional args, --max-tokens) |
| "I'll use WebSearch as fallback" | User asked for bx specifically; fallback violates intent |
| "Goggles uses site: operators" | `site:` is for query strings; Goggles DSL uses `$boost=,site=` format |
| "Goggles syntax is obvious" | DSL has specific format: `$boost=N,site=DOMAIN` per line |
| "Token budget flag is --budget" | Wrong flag name - it's `--max-tokens` (see Common Mistakes) |
| "I don't need token budget" | Unbounded output wastes tokens, can exceed limits |
| "I'll check errors later" | Exit codes require immediate handling; rate limits need backoff |
