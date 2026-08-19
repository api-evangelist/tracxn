---
name: Map a sector landscape and its funding activity on Tracxn
description: >-
  Use Tracxn's two-level sector taxonomy (Practice Area above Feed) to scope a market, then size it
  with the /metric aggregation endpoints instead of paginating — the difference between a handful
  of calls and thousands of credits.
api: postman/tracxn-api-production.postman.json
surfaces: [rest, mcp]
mcp_tools: [search_sectors, search_companies, search_funding_rounds]
operations:
  - POST /api/3.0/practiceareas
  - POST /api/3.0/feeds
  - POST /api/3.0/companies
  - POST /api/3.0/fundingrounds
  - POST /api/3.0/companies/metric
  - POST /api/3.0/fundingrounds/metric
  - POST /api/3.0/acquisitions/metric
generated: '2026-08-14'
method: generated
source: >-
  Grounded in endpoints and example request bodies published in Tracxn's official public Postman
  collection (postman/tracxn-api-production.postman.json), plus the sector-filter guidance in
  https://help.tracxn.com/en/articles/11142764-miscellaneous-faqs-on-tracxn-apis.
---

# Map a sector landscape and its funding activity

## The taxonomy is two levels

**Practice Area** sits above **Feed**. Fintech is a practice area; the specific sub-markets under
it are feeds. Tracxn publishes 3,000+ feeds.

Look them up with `POST /api/3.0/practiceareas` and `POST /api/3.0/feeds`. Business models are
queried through the feeds endpoint with a `dataset` discriminator:

```json
POST https://platform.tracxn.com/api/3.0/feeds
{
  "dataset": "query",
  "filter": {
    "name": "Cybersecurity",
    "publishStatus": ["PUBLISHED"],
    "compsCurationStatus": ["ENABLED"]
  }
}
```

You can **filter by** the taxonomy but you cannot **export** it. Tracxn treats the taxonomy as
proprietary IP and deliberately does not expose the tree.

### Use `feedName`, not `feedId`

Tracxn's own guidance is explicit: when building a payload, prefer `feedName` over `feedId` or
`practiceArea`. Using an unsupported key returns an `InvalidField` error. For practice areas the
correct key is `practiceArea`.

## Size the market with /metric — do not paginate

This is the important instruction in this skill.

If you want counts, use the aggregation endpoints. They take a filter, a `metric`, and a `bucket`
list, and return grouped numbers in one call:

```json
POST https://platform.tracxn.com/api/3.0/fundingrounds/metric
{
  "filter": {
    "practiceAreaId": ["5409cb72e4b0efef66413044"],
    "country": ["India"],
    "companyStage": ["Funded"]
  },
  "metric": "count",
  "bucket": [ { "field": "feedId" } ]
}
```

The alternative — paginating `/companies` 20 records at a time to count them — costs **1 credit per
entity returned**. Counting a 5,000-company market by enumeration costs 5,000 credits. The
aggregation call costs a call.

**But the Key Metrics endpoints are rate-limited 100x tighter than everything else: 10 calls per
hour and 100 per day**, against 10,000/hour for the general API. Budget them deliberately. Decide
your buckets before you call, not by trial and error.

Available aggregations: `/api/3.0/companies/metric`, `/fundingrounds/metric`, `/investors/metric`,
`/acquisitions/metric`.

## Then enumerate only what you need

Once the aggregate tells you where the mass is, pull the actual companies for the buckets that
matter:

```json
POST https://platform.tracxn.com/api/3.0/companies
{
  "filter": { "feedName": ["Cybersecurity"], "country": ["India"], "companyStage": ["Funded"] },
  "sort": {},
  "from": 0,
  "size": 20
}
```

Walk `from` in steps of 20 — `size` cannot exceed 20.

## Funding activity over time

`POST /api/3.0/fundingrounds` supports date filters on **event dates** (funding round date, news
date). There is no filter for "records changed since X" — Tracxn's change detection is
poll-and-compare, because **there are no webhooks**. The provider states plainly that server-side
push does not exist and is only on the roadmap.

Two more filter facts worth knowing:

- There is no direct "latest round amount" filter. Total Funding reflects all-time funding, not the
  most recent round; combine it with `latestFundingDate` to bound the window.
- To find near-unicorns, use the Unicorn & Soonicorn Status filter with value `SOONICORN`.

## If you are on MCP instead

`search_sectors` covers both practice areas and feeds. But note the gap: **no MCP tool returns
aggregates.** On MCP you can only enumerate, which means paying credits per result. For market
sizing, REST is the correct surface.
