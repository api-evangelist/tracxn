---
name: Connect an AI client to the Tracxn MCP server
description: >-
  Point any MCP-compatible client at Tracxn's hosted server, complete the OAuth flow, and work
  within the credit model — including what to do when tools appear but queries fail.
api: mcp/tracxn-mcp.yml
surfaces: [mcp]
mcp_tools: [load_prompts, resolve_entities, search_companies, search_legal_entities, search_funding_rounds, search_acquisitions, search_investors, search_ipo_events, search_time_series, search_locations, search_sectors]
operations: []
generated: '2026-08-14'
method: generated
source: >-
  Grounded in Tracxn's published MCP documentation
  (https://help.tracxn.com/en/articles/14686877-what-is-tracxn-mcp,
  .../15131843-connect-any-other-mcp-compatible-client,
  .../15131870-available-data-tools-with-tracxn-mcp, .../16049141-tracxn-mcp-credits,
  .../15131917-security-data-access-troubleshooting) and a live probe of
  https://platform.tracxn.com/mcp on 2026-08-14.
---

# Connect an AI client to the Tracxn MCP server

Tracxn runs a **remote** MCP server. There is nothing to install — no npx package, no local
process. A client is pointed at one URL and the user authenticates in a browser.

## The endpoint

```
https://platform.tracxn.com/mcp
```

Generic client configuration, verbatim from Tracxn's documentation:

```json
{
  "mcpServers": {
    "tracxn": {
      "url": "https://platform.tracxn.com/mcp"
    }
  }
}
```

That is the whole configuration. The client discovers everything else: the server advertises
RFC 9728 protected-resource metadata and RFC 8414 authorization-server metadata, supports dynamic
client registration, and requires PKCE (S256). Any compliant MCP client can therefore connect
without Tracxn-specific setup.

## Prerequisites

- An active **Tracxn subscription with MCP access enabled**. This is not self-serve — it is
  available on request via the platform's AI integrations page.
- Tracxn login credentials for the browser OAuth step.

Officially supported: Claude (web and desktop), ChatGPT, Cursor, Gemini CLI, Perplexity — including
on free tiers of those clients. Also works with Cline, Continue, Windsurf and anything built on the
MCP SDK.

An **API-token** method for in-house LLM platforms and Microsoft Copilot Studio is documented as
*coming soon*. It is not available yet; OAuth is the only working method.

## First connection

The client detects the OAuth requirement and opens a browser. The user logs in to Tracxn once, and
the client stores the token.

The scope granted is a single `read`. There is no write scope because there is no write surface —
every tool is a query.

## The 11 tools

| Tool | Retrieves |
|---|---|
| `load_prompts` | Tracxn's own instructions and prompt templates |
| `search_companies` | Company details by name or domain |
| `search_legal_entities` | Legal entity details linked to companies |
| `search_funding_rounds` | Funding history, amounts, investors |
| `search_acquisitions` | Acquisition records and deal details |
| `search_investors` | Investor profiles and portfolio data |
| `search_ipo_events` | IPO events and public market history |
| `search_time_series` | Historical financials and employee growth |
| `search_locations` | Companies filtered by geography |
| `search_sectors` | Companies filtered by industry sector |
| `resolve_entities` | A company website → Tracxn's internal entity id |

Call `load_prompts` early — it is Tracxn's own guidance on how to drive the rest, and it is free.

`resolve_entities` is the one to reach for whenever you have a domain and need a record. Everything
else is keyed on Tracxn's internal id.

## Credits — 1 per result

**Every result returned costs 1 MCP credit.** A company, a funding round, an investor, an
acquisition: each counts as one.

- Asking for 50 companies costs 50 credits.
- A company's 10 funding rounds cost 10 credits.
- A broad "build me a market analysis" prompt costs credits for every result across every step it
  takes — this is where allocations disappear.
- Status checks ("am I connected?") are **free**.
- Queries returning nothing are **free**.

Each user has a monthly cap, typically **2,000–10,000** depending on plan. When you hit it, access
resumes automatically at the next monthly reset; there is a "Request for More Credits" button on
the usage page.

So: narrow the filter before you ask. Ask for the companies in one sector in one country at one
stage, not "all fintech companies".

## Troubleshooting

**"Not authorized" / 401** — the Tracxn session expired. Disconnect and reconnect. Also confirm the
plan actually includes MCP access.

**"No tools available" after connecting** — restart the client; some need a restart after the first
OAuth completion. Confirm the browser redirect actually returned to the client.

**Tools appear but queries error** — this is an entitlement problem, not an auth problem.
Authentication and authorization are separate here: a valid `read` token does not imply access to
every dataset, and financials in particular may require a higher tier. Try a simple query first
("find Razorpay on Tracxn") to isolate connectivity from entitlement.

**Connection failed / invalid URL** — check for a trailing space in the URL, and note that
corporate firewalls may block MCP endpoints; the Tracxn MCP domain may need whitelisting.

## Security posture

Tracxn never sees the AI client's credentials, and the AI client never sees the Tracxn password —
that is what the OAuth indirection buys. All queries are processed on Tracxn's servers, and no
Tracxn data is retained by the AI client beyond the conversation. Revoke by disconnecting the
integration in the client.

One account can be connected to several AI clients at once, and each team member connects with
their own credentials.

## What MCP cannot do

There is no aggregation tool. Counting a market over MCP means enumerating it and paying per
result. If you need counts, use the REST `/metric` endpoints instead — see
`skills/tracxn-sector-landscape.md`.
