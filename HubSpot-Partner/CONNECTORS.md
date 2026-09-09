# Connectors

This document lists all connectors required or optionally used by this plugin's skills.

## Bundled connector

| Connector | Type | Endpoint |
|-----------|------|----------|
| HubSpot MCP | Bundled (via `.mcp.json`) | `https://mcp.hubspot.com/anthropic` |

The HubSpot MCP server is the plugin's primary connector. It is configured in `.mcp.json` and provides all CRM tools (search, object read/write, properties, organization and owner details). Every skill in this plugin reads the **PARTNER_CLIENT** CRM object (object type `0-145`) through this connector.

### Authentication (OAuth)

`.mcp.json` ships the `hubspot` server with no `oauth` block, mirroring the HubSpot Sales plugin and Anthropic's reference plugin (`anthropics/knowledge-work-plugins`). Authentication relies on the OAuth client the HubSpot MCP server itself advertises (DCR / PKCE).

The authenticated portal must be a **HubSpot Solutions Partner portal** — PARTNER_CLIENT records only exist in partner portals, and every query is implicitly scoped to the logged-in partner's portal via `get_organization_details`.

## Optional connectors

None. All skills in this plugin operate entirely on HubSpot CRM data. Draft-outreach and talking-point features generate text in chat — they do not require an email or calendar connector.
