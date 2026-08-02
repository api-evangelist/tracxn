# Tracxn

Tracxn ([tracxn.com](https://tracxn.com)) is a market intelligence platform for private company data — 7.7M+ companies across 2,000+ sectors and 3K+ feeds — used by VC, PE, investment banking and corporate M&A teams. Founded in Bengaluru, backed by Accel, and listed on the Indian stock exchanges (NSE: TRACXN).

Programmatic access ships as Data Solutions: the Tracxn API v2.2 (JSON-over-HTTPS POST endpoints at `platform.tracxn.com/api/2.2` for companies, investors, funding transactions and acquisitions, with a rate-limited Playground sandbox), an official Tracxn MCP server for AI assistants (Claude, ChatGPT, Gemini, Cursor), scheduled SFTP dumps, and Snowflake/BigQuery integrations.

Artifacts in this repo: MCP server profile (`mcp/`), authentication profile (`authentication/`), sandbox/Playground (`sandbox/`), API conventions (`conventions/`), error catalog (`errors/`), lifecycle (`lifecycle/`), conformance (`conformance/`), data model (`data-model/`), llms.txt (`llms/`), and a domain security probe (`security/`). See `apis.yml` for the APIs.json index.
