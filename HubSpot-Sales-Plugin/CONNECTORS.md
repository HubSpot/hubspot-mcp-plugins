# Connectors

This document lists all connectors required or optionally used by this plugin's skills.

## Bundled connector

| Connector | Type | Endpoint |
|-----------|------|----------|
| HubSpot MCP | Bundled (via `.mcp.json`) | `https://mcp.hubspot.com/anthropic` |

The HubSpot MCP server is the plugin's primary connector. It is configured in `.mcp.json` and provides all CRM tools (contacts, companies, deals, properties, search, etc.).

### Authentication (OAuth)

`.mcp.json` ships the `hubspot` server with no `oauth` block, mirroring Anthropic's reference plugin (`anthropics/knowledge-work-plugins`). Authentication relies on the OAuth client the HubSpot MCP server itself advertises (DCR / PKCE).

## Optional connectors

| Connector | Purpose | Category placeholder |
|-----------|---------|----------------------|
| Gmail | Pull contacts from email threads; log email history against CRM records | `~~email` |

The `import-contacts` skill detects Gmail at runtime and offers it as a source if connected. The skill is fully functional without Gmail — manual CSV/paste import works independently.
