# hubspot-mcp-plugins

Official HubSpot plugins for Claude Code, powered by the [HubSpot MCP Server](https://mcp.hubspot.com).

## Available Plugins

| Plugin | Description |
|--------|-------------|
| **HubSpot Sales** | CRM tools for managing contacts, deals, and onboarding workflows |
| **HubSpot Partner** | Solutions Partner tools for managing your book of business across PARTNER_CLIENT records |

## Installation

### Claude Code (CLI)

In Claude Code, add the HubSpot marketplace:

```text
/plugin marketplace add HubSpot/hubspot-mcp-plugins
```

Then run the command for the plugin you want.

**HubSpot Sales**

```text
/plugin install hubspot-sales@hubspot-claude-plugins
```

**HubSpot Partner**

```text
/plugin install hubspot-partner@hubspot-claude-plugins
```

### Claude Desktop App

1. Open **Settings** > **Plugins**
2. Click **Add** > **Add marketplace**
3. Paste the repository URL: `https://github.com/HubSpot/hubspot-mcp-plugins`
4. Select the plugin you want to install
