---
name: brave-search-cli-usage
description: Use when needing to use Brave Search CLI (bx) for web searches, RAG grounding, AI answers, or research automation. Covers correct syntax, Goggles patterns, API key setup, and command selection.
---

# Brave Search CLI Usage

## Overview
Brave Search CLI (`bx`) is a zero-dependency CLI for Brave Search API, purpose-built for AI agents and LLMs. It replaces the search → scrape → extract pipeline with pre-extracted, token-budgeted content.

**Free Plan Limitation:** The free plan only supports the `web` command. Commands `context`, `answers`, and `news` require paid plans.

**Key insight for free plan:** Use `bx web` with `--count` to control result volume. Process JSON output with `jq` or Python to extract specific data. Use `site:` operators in queries and `--include-site`/`--exclude-site` or `--goggles` for domain filtering.

## When to Use (Free Plan)

**Use bx web when:**
- You need current information beyond training data
- Researching errors, APIs, or code patterns
- Gathering documentation for code generation
- Performing competitive analysis or security research
- Searching for financial data, prices, or market information
- You need to filter by specific sites using `site:` operators

**Don't use bx when:**
- The information is in your training dataset (pre-2024) - search not needed
- You're offline (bx requires network)
- You need AI-synthesized answers (requires `answers` command - paid plan)
- You need pre-extracted article content (requires `context` command - paid plan)
- You need news-only filtering (requires `news` command - paid plan)

## Core Pattern: Correct Command Syntax (Free Plan)

### ❌ Common Mistakes (from baseline tests)

```bash
# WRONG: --query flag doesn't exist
bx --query "search term"

# WRONG: --max-tokens is not supported in free plan web command
bx web "search" --max-tokens 4096

# WRONG: news/answers/context not available on free plan
bx context "search"
bx answers "question"
bx news "topic"
```

### ✅ Correct Patterns (Free Plan - web only)

```bash
# Simple web search
bx "your search query"

# Web search with result count limitation
bx "your search query" --count 5

# Web search with site operator in query
bx "site:docs.rs axum middleware" --count 5

# Web search with domain filtering (include only)
bx "axum middleware" --include-site docs.rs --include-site github.com --count 5

# Web search with domain filtering (exclude)
bx "javascript tutorial" --exclude-site w3schools.com --count 10

# Web search with Goggles DSL for advanced ranking (free plan supports this)
bx "serde custom deserializer" \
  --goggles '$boost=5,site=docs.rs
$boost=5,site=crates.io
$boost=3,site=github.com
$discard,site=geeksforgeeks.org' \
  --count 5
```

**FREE PLAN LIMITATIONS:**
- Only `web` command is available
- `--max-tokens` flag is NOT supported (use `--count` to limit results)
- `context`, `answers`, `news` commands require paid plans
- `--freshness` flag is for `news` command only (not available)

## Quick Reference (Free Plan)

| Command | Use Case | Output Shape | Typical Flags | Plan |
|---------|----------|--------------|---------------|------|
| `web` | Full search with operators | `.web.results[]` | `--count`, `--goggles`, `--include-site`, `--exclude-site` | ✅ Free |
| `context` | RAG/grounding (pre-extracted) | `.grounding.generic[].snippets[]` | `--max-tokens`, `--threshold`, `--goggles` | ❌ Paid |
| `answers` | AI-generated explanation | OpenAI-compatible streaming | `--no-stream`, `--model` | ❌ Paid |
| `news` | News articles only | `.results[]` | `--freshness`, `--count` | ❌ Paid |

**Exit codes:** 0=success, 1=client error, 2=usage error, 3=auth error, 4=rate limited, 5=server error

**Free Plan JSON Output Structure:**
```json
{
  "type": "search",
  "query": {...},
  "web": {
    "results": [
      {
        "title": "...",
        "url": "...",
        "description": "...",
        "profile": {...}
      }
    ]
  }
}
```

## Goggles: Custom Re-Ranking

**IMPORTANT DISTINCTION:**

| Method | Purpose | Syntax | Where to Use |
|--------|---------|--------|--------------|
| `site:` operators | Filter results in query | `site:docs.rs axum` | Inside query string (works with any command) |
| Goggles DSL | Re-rank/boost/discard | `$boost=5,site=docs.rs` | In `--goggles` flag (one rule per line) |
| `--include-site` | Easy domain allowlist | `--include-site docs.rs` | As flag (multiple allowed) |
| `--exclude-site` | Easy domain blocklist | `--exclude-site w3schools.com` | As flag (multiple allowed) |

**Do NOT mix them:** `--goggles "site:docs.rs"` is **WRONG**. Use `$boost=5,site=docs.rs` in Goggles.

### Inline Rules (Zero Setup) - Goggles DSL (Free Plan)

```bash
# Boost official Rust sources, discard tutorial spam
bx web "serde custom deserializer" \
  --goggles '$boost=5,site=docs.rs
$boost=5,site=crates.io
$boost=3,site=github.com
$discard,site=geeksforgeeks.org
$discard,site=w3schools.com' \
  --count 5
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

### From a File (Reusable Goggles) - Free Plan

```bash
# Create reusable goggles file
cat > .goggle << 'EOF'
$boost=5,site=docs.rs
$boost=3,site=github.com
/blog/$downrank=3
$discard,site=geeksforgeeks.org
EOF

# Reuse across queries with free plan
bx web "tokio async" --goggles @.goggle --count 5
bx web "serde serialize" --goggles @.goggle --count 5
```

## Controlling Output Size (Free Plan)

On the free plan, use `--count` to limit the number of search results:

```bash
# Get 5 results (reduces output size)
bx "your query" --count 5

# Get 10 results (default may be higher)
bx "your query" --count 10

# Combine with domain filtering for focused results
bx "query" --include-site docs.rs --count 5
```

**Note:** Free plan does NOT support `--max-tokens`, `--max-tokens-per-url`, `--max-urls`, or `--threshold` flags. Those are paid plan features for `context` command. Use `--count` to manage result volume on free plan.

**Extracting Specific Data:**
Since `web` returns JSON, pipe to `jq` or Python to extract specific fields:

```bash
# Extract just titles and URLs using jq
bx "query" --count 5 | jq '.web.results[] | {title, url}'

# Extract descriptions with Python
bx "query" --count 3 | python3 -c "import sys, json; data = json.load(sys.stdin); [print(r.get('description', 'N/A')) for r in data.get('web', {}).get('results', [])]"
```

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

## Command Selection Guide (Free Plan)

| Your Need | Command & Strategy | Why |
|-----------|-------------------|-----|
| General web search | `bx "query" --count N` | Only free plan option |
| Search specific site | `bx "site:docs.rs query" --count N` | `site:` operator works in `web` |
| Filter to trusted sources | `bx "query" --include-site docs.rs --include-site github.com --count N` | Domain allowlist |
| Exclude low-quality sites | `bx "query" --exclude-site w3schools.com --count N` | Domain blocklist |
| Advanced ranking | `bx "query" --goggles '@file' --count N` | Boost/discrank/discard rules |
| Extract structured data | `bx "query" \| jq '.web.results[]'` | JSON output for parsing |

**Paid Plan Features (Not Available on Free):**
- `context`: Pre-extracted article content with token budgeting
- `answers`: AI-synthesized explanations with citations
- `news`: News-only results with freshness filters
- `--max-tokens`, `--threshold`, `--result-filter`: Advanced result control

## Implementation Checklist (Free Plan)

**Before running bx:**
- [ ] API key configured? (`bx config show-key` to verify)
- [ ] bx installed? (`which bx` or `bx --version`)
- [ ] Using `web` command? (only free option)
- [ ] Set result count? (use `--count N` to limit output)
- [ ] Need domain filtering? (`--include-site`, `--exclude-site`, or `--goggles`)
- [ ] Plan to parse JSON? (need `jq` or Python for extraction)

**Constructing queries (free plan):**
- [ ] Query is specific enough (include error messages, names, version numbers)
- [ ] Use `site:` operator in query string for domain restriction
- [ ] Exclude low-quality sites proactively (`--exclude-site`)
- [ ] Test simple query first, then add filters if needed
- [ ] Pipe to `jq` if you need structured data extraction

## Common Mistakes (Free Plan)

| Mistake | Why It's Wrong | Fix |
|---------|----------------|-----|
| `bx --query "term"` | `--query` flag doesn't exist | Use positional arg: `bx "term"` |
| Using `context`/`answers`/`news` | Not available on free plan | Use `web` command only |
| `bx web --max-tokens 4096` | `--max-tokens` not supported | Use `--count N` to limit results |
| Forgetting `--count` | May get many results, slow response | Add `--count 5` or appropriate number |
| `--goggles "site:docs.rs"` | `site:` is query operator, not Goggles DSL | Use `$boost=5,site=docs.rs` in Goggles |
| Using `--freshness` | Only for `news` command (paid) | Not available on free plan |
| Expecting pre-extracted content | Free plan only gives search results | Parse JSON output for data |
| Not checking exit codes | Silent failures ignored | Check `$?` after bx commands |
| Using `--api-key` in scripts | Key visible in process list | Use `bx config set-key` or env var |
| Assuming `web` includes news only | `web` mixes web + news by default | Filter results manually if needed |

## Real-World Workflows (Free Plan)

### Debugging an Error (Free Plan)

```bash
# 1. Broad web search with limited results
bx "TypeError cannot unpack non-iterable NoneType" --count 5

# 2. Narrow to Stack Overflow
bx "site:stackoverflow.com TypeError cannot unpack" --count 5

# 3. Extract specific info from results
bx "TypeError cannot unpack non-iterable NoneType" --count 5 | jq '.web.results[] | {title, url, description}'
```

### Researching a Technology/Rust Crate (Free Plan)

```bash
# Search with domain filtering and Goggles
bx "reqwest crate security vulnerabilities" \
  --goggles '$boost=5,site=docs.rs
$boost=5,site=crates.io
$boost=3,site=github.com
$discard,site=geeksforgeeks.org' \
  --count 5

# Extract titles and URLs for manual review
bx "reqwest crate maintenance status 2026" --count 5 | jq '.web.results[] | {title, url}'
```

### Checking Current Prices/Financial Data (Free Plan)

```bash
# Search for current market prices
bx "WTI crude oil price today" --count 3
bx "Brent crude oil futures price" --count 3

# Process results to extract price data
bx "current BTC price USD" --count 5 | python3 -c "
import sys, json, re
data = json.load(sys.stdin)
for r in data.get('web', {}).get('results', []):
    desc = r.get('description', '')
    # Look for price patterns
    prices = re.findall(r'\$[\d,]+\.?\d*', desc)
    if prices:
        print(f\"{r['title']}: {prices[0]}\")
"
```

### Pre-Project Research (Free Plan)

```bash
# Search for framework/library information
bx "React 19 new features migration guide" --count 8

# Focus on official docs
bx "React 19 upgrade guide site:react.dev" --count 5

# Exclude tutorial spam
bx "learn React 19" --exclude-site medium.com --exclude-site dev.to --count 5
```

### Monitoring Market/News (Free Plan)

```bash
# Broad search for recent developments
bx "AI regulation 2026 latest" --count 10

# Filter to reputable sources
bx "AI policy developments" \
  --include-site whitehouse.gov \
  --include-site ec.europa.eu \
  --include-site reuters.com \
  --count 8
```

---

## JSON Output Processing (Free Plan)

Since `bx web` returns structured JSON, use `jq` or Python to extract specific data:

```bash
# Extract all titles and URLs with jq
bx "query" --count 5 | jq '.web.results[] | {title, url}'

# Extract just descriptions
bx "query" --count 3 | jq -r '.web.results[].description'

# Count number of results
bx "query" --count 10 | jq '.web.results | length'

# Filter results by domain
bx "query" --count 10 | jq '.web.results[] | select(.url | contains("github.com"))'

# Format as markdown links
bx "rust tutorial" --count 5 | python3 -c "
import sys, json
data = json.load(sys.stdin)
for r in data.get('web', {}).get('results', []):
    print(f\"* [{r['title']}]({r['url']})\")
"

# Extract and search within descriptions for patterns (e.g., prices)
bx "current bitcoin price" --count 5 | python3 -c "
import sys, json, re
data = json.load(sys.stdin)
for r in data.get('web', {}).get('results', []):
    desc = r.get('description', '')
    prices = re.findall(r'\$[\d,]+', desc)
    if prices:
        print(f\"{r['title']}: {prices[0]}\")
"
```

**Recommended Tools:**
- `jq` - Lightweight command-line JSON processor (install: `apt install jq`, `brew install jq`)
- `python3 -c` - Python one-liners (usually pre-installed)
- `jq` is faster for simple extraction; Python better for complex parsing

- Using `--query` flag → Doesn't exist; use positional arg `bx "query"`
- Using `context`/`answers`/`news` → Not available on free plan; use `web` only
- Using `--max-tokens` → Not supported on free plan; use `--count` instead
- Forgetting `--count` → May get many results; slower response and harder to parse
- Not checking `bx` exists → `which bx || echo "install required"`
- Ignoring exit codes → Silent failures; always check `$?`
- Passing API key via `--api-key` in scripts → Visible in process list; use `bx config set-key` or env var
- Expecting full article content → Free plan only provides search snippets
- Using `--freshness` → Only for `news` command (paid plan)
- Misusing Goggles DSL → `site:` goes in query; `$boost=,site=` goes in `--goggles`

## Rationalization Table (Free Plan)

| Excuse | Reality |
|--------|---------|
| "I can guess the syntax" | bx has specific conventions; guessing causes errors (see Common Mistakes) |
| "Other search tools use similar flags" | bx uses positional args, not `--query`; `--count` not `--max-tokens` |
| "I'll use WebSearch as fallback" | User asked for bx specifically; fallback violates intent |
| "Goggles uses site: operators" | `site:` is for query strings; Goggles DSL uses `$boost=,site=` format |
| "Goggles syntax is obvious" | DSL has specific format: `$boost=N,site=DOMAIN` per line |
| "I need token budget" | Free plan doesn't support `--max-tokens`; use `--count` to limit results |
| "I'll check errors later" | Exit codes require immediate handling; rate limits need backoff |
| "context/answers are better" | Not available on free plan; make the most of `web` with filters |
| "I need full article text" | Free plan only returns search results; visit URLs manually or upgrade |
| "I don't need to parse JSON" | `bx web` outputs JSON; need `jq`/Python to extract structured data |
