<!-- mcp-name: io.github.aion-autonomous-labs/edgar-answers-mcp -->

# EDGAR Answers — SEC filings for AI agents

Give your agent **answers** from SEC EDGAR, not raw API responses.

> This repository holds the documentation and registry manifest for a **hosted**
> MCP server. There is nothing to install or run: connect to the endpoint below
> from any MCP client. The server itself is operated as a managed service.

Every US public company files its financials with the SEC, for free, in XBRL.
The catch is that "free and public" is not the same as "usable": companies tag
the same line item a dozen different ways, annual and quarterly figures live in
the same stream, restatements silently overwrite history, and Form 4 insider
filings arrive as raw XML. Existing EDGAR MCP servers hand your agent that mess
and wish it luck.

This server does the parsing first. Your agent asks for revenue; it gets
revenue.

## Tools

### `get_financials` — normalized annual statements

Multi-year income statement, balance sheet, and cash flow for any US-listed
company, from a ticker or CIK.

```
get_financials(company: "AAPL", years: 4)
→ revenue, grossProfit, operatingIncome, netIncome, epsDiluted,
  operatingCashFlow, capex, totalAssets, totalLiabilities,
  stockholdersEquity, cashAndEquivalents, longTermDebt, sharesDiluted
  — one clean row per fiscal year
```

Behind that: an ordered tag-fallback chain per line item (filers disagree on
which us-gaap tag means "revenue"), duration filtering so annual figures aren't
contaminated by quarterly cumulatives, and restatement resolution to the most
recently filed value. Unreported items come back `null` rather than guessed.

### `filing_diff` — what changed year over year

```
filing_diff(company: "NVDA", section: "risk_factors")
→ paragraphs added and removed between the two most recent 10-Ks
```

The question an analyst actually asks — *what did they start saying this year,
and what did they quietly drop?* — answered at paragraph level, with
boilerplate that merely reflowed filtered out.

### `insider_activity` — Form 4, parsed

```
insider_activity(company: "MSFT", filings: 10)
→ who, role, transaction type, shares, price, holdings after — tidy rows
```

Open-market buys separated from tax withholding and option exercises, so your
agent doesn't mistake a scheduled vest for a conviction purchase.

### `filing_section` — any section as clean text

```
filing_section(company: "TSLA", section: "risk_factors" | "mdna" | "business" | "1A")
```

Table-of-contents decoys are filtered out; you get the real section body.

### `search_filings` — full-text search since 2001

```
search_filings(query: "\"supply chain disruption\"", forms: "10-K", startDate: "2025-01-01")
```

## Connect

Streamable HTTP endpoint:

```
https://aion-org--edgar-answers-mcp.apify.actor/mcp
```

Authenticate with your Apify API token as a bearer token. In an MCP client
config:

```json
{
  "mcpServers": {
    "edgar-answers": {
      "url": "https://aion-org--edgar-answers-mcp.apify.actor/mcp",
      "headers": { "Authorization": "Bearer YOUR_APIFY_TOKEN" }
    }
  }
}
```

For clients that only speak stdio, bridge with `npx mcp-remote <url>
--header "Authorization: Bearer YOUR_APIFY_TOKEN"`.

The server runs in standby mode: it scales to zero when idle and wakes on your
first request, so the first call after a quiet period takes a few seconds.

## Pricing

Pay per successful call — failed calls are not billed.

| Call | Price |
|---|---|
| `search_filings` | $0.005 |
| `get_financials` | $0.02 |
| `insider_activity` | $0.02 |
| `filing_section` | $0.03 |
| `filing_diff` | $0.05 |

## Data and compliance

Source data is SEC EDGAR — US government filings, public domain. Requests are
made with a declared User-Agent and rate-limited well under the SEC's published
ceiling, per their [access guidelines](https://www.sec.gov/os/accessing-edgar-data).
Nothing here is scraped from a site that prohibits it.

## Limits, stated plainly

- **US GAAP filers.** Foreign private issuers reporting under IFRS return no
  normalized financials; `get_financials` says so rather than guessing.
- **Coverage follows EDGAR.** Full-text search reaches back to 2001; XBRL
  financials exist from roughly 2009 onward.
- **Not investment advice.** This is filing data, faithfully parsed. What you
  conclude from it is yours.
- `filing_diff` compares the two most recent filings of a form; arbitrary
  filing pairs aren't exposed yet.

Found a parsing bug or want a tool that isn't here? Open an issue here or on
the Actor — specific reports about specific tickers get fixed.

---

Listed on the [Apify Store](https://apify.com/aion_org/edgar-answers-mcp).
Documentation in this repository is © Aion; the service is proprietary.
