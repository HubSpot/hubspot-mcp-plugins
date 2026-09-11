# Connectors

This document lists all connectors required or optionally used by this plugin's skills.

## Bundled connector

This plugin bundles the **hubspot** connector in `.mcp.json`, referenced by name rather than by URL. It provides all CRM tools (contacts, companies, deals, properties, search, etc.) that this plugin's skills use.

## Optional connectors

**Gmail** can optionally be connected to pull contacts from email threads and log email history against CRM records. The `import-contacts` skill detects Gmail at runtime and offers it as a source if connected, but works fully without it — manual CSV/paste import is always available.
