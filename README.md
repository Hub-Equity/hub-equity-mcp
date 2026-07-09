<!-- mcp-name: com.hub-equity/hub-equity-mcp -->

# hub-equity-mcp

**Standardized XBRL financial data for LLM agents.** A Model Context Protocol (MCP)
server that exposes normalized financial facts from US SEC (EDGAR) and European
ESEF filings to Claude Desktop, Cursor, and any MCP-aware client.

Hub-Equity is the first MCP server to serve standardized **European ESEF** filings
alongside US SEC data through one consistent hub-concept vocabulary, so an agent
can ask for `REVENUE` or `TOTAL_ASSETS` and get a comparable, source-linked value
whether the issuer files with the SEC or under ESEF.

> **Maturity.** The data engine and public REST API behind this connector run in
> production and power Hub-Equity's own chat. This PyPI package is the newly
> published client for that API; the surface is stable (SemVer 1.0), but as a
> distributed package it is fresh, hence the Beta classifier.

## Why

- **One vocabulary across two regimes.** SEC us-gaap and ESEF ifrs-full concepts
  are mapped to a single set of standardized hub codes, so cross-issuer and
  cross-taxonomy comparison works out of the box.
- **Every number is source-linked.** Facts carry their filing, period, and
  provenance so an agent can cite rather than guess.
- **Read-only and closed-world.** Every tool advertises `readOnlyHint=true`,
  `idempotentHint=true`, `destructiveHint=false`, `openWorldHint=false` per the
  MCP spec, so clients can reason about safety and caching without introspection.
- **No database credentials.** The published package talks only to the public
  REST API (`https://api.hub-equity.com`) over HTTPS. It never ships or requires
  a Supabase or DB key.

## Install

```bash
pip install hub-equity-mcp
```

Requires Python 3.12 or newer.

## Authentication and access

The server talks only to the public REST API. Two modes:

- **Anonymous (no key).** Works out of the box, no account required. Gives the
  base tool set (entity search, normalized facts, time series, segments,
  screener, FX conversion, and more), capped at **60 requests per minute** per IP.
- **With a `hubq_` key** (env var `HUBEQUITY_API_KEY`). Unlocks the Pro tools
  (restatement diffs, calculation trees, cross-period compare, data-quality
  grades, validation checks, extension concepts). On the Pro plan a key also
  raises the limit to **300 requests per minute** (1000 on Enterprise). A key is
  free to create from a Hub-Equity account (beta). Paid Pro and Enterprise plans
  exist but are not billed at this stage.

The published package never reaches the database directly, only the REST API.

## Configure your client

### Claude Desktop

Add to `claude_desktop_config.json`
(macOS: `~/Library/Application Support/Claude/claude_desktop_config.json`,
Windows: `%APPDATA%\Claude\claude_desktop_config.json`):

```json
{
  "mcpServers": {
    "hub-equity": {
      "command": "python",
      "args": ["-m", "hub_equity_mcp.server"],
      "env": {
        "HUBEQUITY_API_KEY": "hubq_live_..."
      }
    }
  }
}
```

Omit `HUBEQUITY_API_KEY` to run anonymously (60 requests per minute, base tools only).

### Cursor

Add to `.cursor/mcp.json` (project root) or the global Cursor MCP settings:

```json
{
  "mcpServers": {
    "hub-equity": {
      "command": "python",
      "args": ["-m", "hub_equity_mcp.server"],
      "env": {
        "HUBEQUITY_API_KEY": "hubq_live_..."
      }
    }
  }
}
```

### Environment variables

- `HUBEQUITY_API_KEY` (optional): a `hubq_` key for premium tools and the higher
  rate limit. Absent means anonymous mode.
- `HUBEQUITY_API_URL` (optional): defaults to `https://api.hub-equity.com`.
  `https://` is enforced whenever a key is set (the server refuses to send the
  Bearer key over plaintext to a non-loopback host).

## Capabilities

| Type | Count |
|---|---|
| Tools | 19 (13 Free, 6 Pro) |
| Resources | 9 (7 static, 2 URI templates) |
| Prompts | 8 analytical templates |
| Completion API | `{hub_code}` autocomplete |

## Tools

The machine-readable tier catalog is served as a resource
(`hub-equity://catalog/tool-tiers`). Free tools cover discovery and identity;
Pro tools add forensic depth (calculation trees, restatement diffs, cross-period
comparison, quality grades, validation results, extension concepts).

| Tool | Tier | What it does |
|---|---|---|
| `find_entity(query)` | Free | Search by name, ticker, or CIK. Returns the `entity_id` other tools need. |
| `get_fact(entity_id, code, fiscal_year, period_type)` | Free | One normalized value plus its filing source. |
| `get_fact_decomposition(entity_id, code, fiscal_year, depth)` | Free (depth 1) / Pro (depth 2-3) | Hub rollup, XBRL calc-linkbase children, and dimensional breakdown. |
| `search_concept(query)` | Free | Resolve a hub code from a label or XBRL qname. |
| `list_hubs(category?, ...)` | Free | Enumerate the standardized hub catalog by category. |
| `get_entity_profile(entity_id)` | Free | Sector, auditor, employees, fiscal year end, recent filings. |
| `get_metric_history(entity_id, code, n_years)` | Free | N-year time series with YoY growth and CAGR. |
| `get_segments(entity_id, code, fiscal_year)` | Free | Dimensional axis/member breakdown (segment, geography). |
| `get_amendments(entity_id, fiscal_year?)` | Free | 10-K/A restatement summary. |
| `compare_entities(ids, codes, fiscal_year)` | Free (up to 3x5) / Pro (up to 10x10) | Cross-issuer comparison matrix at one period. |
| `roll_up_metric(entity_id, code, fiscal_year)` | Free | Compute a value from signed children when it is not directly tagged. |
| `convert_currency(amount, from, to, date?, rate_type?)` | Free | ECB reference-rate FX conversion (closing, average YTD, average prior year). |
| `screen_companies(filters, sort, limit)` | Free (limit 20, no quality filter) / Pro (higher) | Filter the issuer universe by metadata, revenue, audit, and data quality. |
| `get_amendment_diff(entity_id, fiscal_year?, code?, min_diff_pct)` | Pro | Per-concept restatement diffs with a materiality filter. |
| `compare_filings(entity_id, fy_a, fy_b, codes?)` | Pro | Cross-period same-entity compare with new / removed / sign-flip / restatement flags. |
| `get_filing_calc_tree(filing_id, link_role?, statement?)` | Pro | Full presentation tree of a single filing. |
| `get_extension_concepts(entity_id, status_filter?, limit?)` | Pro | Issuer-specific qnames declared outside standard taxonomies. |
| `get_data_quality_grade(entity_id)` | Pro | A+ to D grade, coverage, freshness, direct-vs-derived breakdown. |
| `get_validation_results(filing_id?, entity_id?, fiscal_year?, status?)` | Pro | XBRL accounting and calculation-linkbase checks. |

## Resources

| URI | Type | Purpose |
|---|---|---|
| `hub-equity://catalog/hubs` | json | Full standardized hub catalog with EN/FR labels and category. |
| `hub-equity://catalog/categories` | json | Hub counts per category. |
| `hub-equity://catalog/tool-tiers` | markdown | Free vs Pro tool catalog and gating conditions. |
| `hub-equity://schema/financial-statements` | markdown | Statement structure and reading rules. |
| `hub-equity://catalog/hub/{hub_code}` | template | Forward catalog entry for one hub. |
| `hub-equity://entity/{entity_id}/profile` | template | Full entity snapshot. |
| `hub-equity://prompts/best-practices` | markdown | System-prompt guidance for client integrations. Load this before calling any tool. |
| `hub-equity://prompts/tool-usage-examples` | markdown | Per-tool few-shot examples (good and anti-pattern). |
| `hub-equity://prompts/data-coverage` | json | Live dataset snapshot (issuer and filing counts, sources, taxonomies, fiscal year range). Cached 24h. |

## Prompts

Eight analytical templates: `peer_comparison`, `quality_of_earnings`,
`restatement_audit`, `sector_overview`, `valuation_screen`,
`goodwill_impairment_risk`, `working_capital_diagnostic`, `cash_flow_consistency`.

## For client developers

Before calling any tool, fetch `hub-equity://prompts/best-practices` and inject
the markdown into your system prompt. This makes your client follow the same tool
routing, source-citation, and numeric-fidelity rules as Hub-Equity's own chat.

```python
# Pseudo-code for a typical MCP client integration
session = mcp.connect("hub-equity-mcp")
best_practices = session.read_resource("hub-equity://prompts/best-practices")
system_prompt = "You are an assistant ...\n\n" + best_practices
# now call session.call_tool("find_entity", {"query": "AAPL"}) etc.
```

## Rate limits

| Mode | Limit | Notes |
|---|---|---|
| Anonymous or Free key | 60 requests / minute | Base tools. |
| Pro key | 300 requests / minute | Unlocks Pro tools. |
| Enterprise key | 1000 requests / minute | Unlocks Pro tools. |

On a 429 the client retries with exponential backoff (up to 3 times) before
raising `RateLimitExceeded`.

## Troubleshooting

| Symptom | Cause | Fix |
|---|---|---|
| `429 Too Many Requests` / `RateLimitExceeded` | Rate cap hit (60/min anon or Free, 300/min Pro) | Add a `HUBEQUITY_API_KEY` on a Pro plan, or slow down the tool-call fan-out. The client already backs off up to 3 times. |
| `HubEquityRestError: HTTP 401` | Invalid or revoked `hubq_` key | Create a new key from your Hub-Equity account settings. |
| `HubEquityRestError: HTTP 403` | Key lacks the scope for a Pro tool | Upgrade the plan, or use the base tool set. |
| Connection or timeout errors | Network issue reaching `api.hub-equity.com`, or a bad `HUBEQUITY_API_URL` | Check connectivity; confirm `HUBEQUITY_API_URL` (if set) points to a reachable `https://` host. |
| `ValueError: HUBEQUITY_API_URL must use https://` | A key is set but the URL is plain `http://` on a non-loopback host | Use `https://`, or unset `HUBEQUITY_API_URL` to fall back to the default API. |
| Server does not appear in Claude Desktop or Cursor | Config JSON error, or `python` not on the client's PATH | Validate the JSON; use an absolute interpreter path if the client cannot resolve `python`. |

## Run locally

```bash
python -m hub_equity_mcp.server
```

Or drive it interactively with the MCP inspector:

```bash
npx @modelcontextprotocol/inspector python -m hub_equity_mcp.server
```

The inspector lists all 19 tools, 9 resources, and 8 prompts and lets you call
each one.

## Development

```bash
pip install -e '.[dev]'
pytest tests/
```

Tests are hermetic: tool tests mock the REST API with `respx`, so no live backend
is needed.

## License

Apache-2.0. See [LICENSE](LICENSE) and [NOTICE](NOTICE). This connector is an open
client to the public Hub-Equity REST API; access to premium data stays gated by
API key, plan, and rate limits on the service side.
