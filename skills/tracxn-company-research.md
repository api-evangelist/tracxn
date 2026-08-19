---
name: Research a private company on Tracxn
description: >-
  Resolve a company by name or domain to its Tracxn entity id, then pull its profile, funding
  history, investors, acquisitions and financial time series — over either the MCP server or the
  REST API, without burning credits on pagination you did not need.
api: postman/tracxn-api-production.postman.json
surfaces: [mcp, rest]
mcp_tools: [resolve_entities, search_companies, search_funding_rounds, search_investors, search_acquisitions, search_time_series]
operations:
  - POST /api/3.0/companies
  - POST /api/3.0/fundingrounds
  - POST /api/2.2/investors
  - POST /api/3.0/acquisitions
  - POST /api/3.0/ipos
  - POST /api/3.0/timeseries/revenue
  - POST /api/3.0/timeseries/valuation
  - POST /api/3.0/timeseries/employeecountannualreport
generated: '2026-08-14'
method: generated
source: >-
  Grounded in the endpoints published in Tracxn's official public Postman collection
  (postman/tracxn-api-production.postman.json) and the MCP tool names published at
  https://help.tracxn.com/en/articles/15131870-available-data-tools-with-tracxn-mcp. No endpoint,
  tool or field below is invented.
---

# Research a private company on Tracxn

Tracxn is a private-market intelligence database. Everything is keyed on an internal 24-character
hex entity id, so the first move in almost every flow is resolving a human identifier (a name or,
better, a domain) to that id.

## Choose the surface first

- **MCP** (`https://platform.tracxn.com/mcp`) — OAuth, conversational, version-agnostic. Best when
  a human is in the loop. Cannot compute aggregates.
- **REST** (`https://platform.tracxn.com/api/3.0`) — `accessToken` header, uniform query-by-POST.
  Best for repeatable pipelines and the only surface with aggregation.

Use **API v3.0**. Version 2.2 still works but Tracxn has announced it is being phased out. Two
things are still v2.2-only: **investor search** (`/api/2.2/investors`) and the bookmarks list.

## Step 1 — resolve the company to an id

On MCP, call `resolve_entities` with the company website. That is exactly what it is for: it maps
an external domain to Tracxn's internal record, and it is the only reliable disambiguator when
several companies share a name.

On REST there is no resolve endpoint. Filter `/api/3.0/companies` by domain and read the id off the
first result.

```json
POST https://platform.tracxn.com/api/3.0/companies
accessToken: <your production token>

{
  "filter": { "domain": ["example.com"] },
  "from": 0,
  "size": 20
}
```

Every request body uses the same four keys — `filter`, `sort`, `from`, `size`. There are no path
or query parameters anywhere in this API.

## Step 2 — pull the profile

Keep the resolved id and reuse it. Every downstream call filters on `companyId`.

Note `lastUpdatedDate` on the company record: it is the only freshness signal in the contract, so
read it before trusting anything time-sensitive.

## Step 3 — funding, investors, exits

```json
POST https://platform.tracxn.com/api/3.0/fundingrounds
{ "filter": { "companyId": ["<id>"] }, "from": 0, "size": 20 }
```

- Acquisitions: `POST /api/3.0/acquisitions`
- IPOs: `POST /api/3.0/ipos` (v3.0 only — there is no v2.2 equivalent)
- Investors: `POST /api/2.2/investors` — still on the old major

## Step 4 — financial time series

Each metric is its own endpoint, all filtered by `companyId`:
`/api/3.0/timeseries/revenue`, `/netprofit`, `/ebitda`, `/valuation`, `/balancesheetassets`,
`/balancesheetequity`, `/balancesheetliability`, `/cashflowoperatingactivity`,
`/cashflowinvestingactivity`, `/employeecountlabourfilings`, `/employeecountannualreport`.

Two traps:

- Values are **raw per year**. Year-over-year growth is not returned — compute it yourself.
- Employee count comes from **two different sources of record** (labour filings vs annual report).
  They can legitimately disagree for the same company. Say which one you used.

## Cost and pagination — read this before looping

`size` **defaults to 20 and maxes out at 20**. There is no way to ask for more. So any result set
above 20 requires walking `from` in steps of 20.

That matters because **billing is per entity returned, not per call**: 1 credit per entity, so
credits are consumed in multiples of 20 as you paginate. A broad filter you paginate to the end is
the single easiest way to burn an allocation. Filter hard first, paginate only as far as you need.

Queries that match nothing cost nothing.

## Error handling

Errors are a custom `{errorCode, message}` envelope — not `problem+json`.

| What you see | What it means | What to do |
|---|---|---|
| 401 | Token missing or invalid | Check the `accessToken` header |
| 403 + `403000000` | Token expired, or entitlement denied | Regenerate the token; confirm the plan covers this dataset |
| 403 + `API out of credits` (code 900) | Allocation exhausted | **Do not retry.** It cannot succeed until Tracxn renews credits |
| 429 | Rate limit exceeded | Back off — but see below |
| `InvalidField` | Unrecognised filter key | Use `feedName` rather than `feedId`/`practiceArea` for sectors |

**There are no rate-limit headers.** No `RateLimit-*`, no `Retry-After`. You cannot read your
remaining budget from a response, so track your own spend. Production allows 100/sec, 1,000/min,
10,000/hour, 100,000/day.

Finally: **missing fields are omitted, not null.** An absent key means Tracxn has no value for it,
not that the call failed.
