# skill-peak-ops

Portfolio operations dashboard over PEAK fault-detection alert tickets in Claude Chat. Returns markdown tables showing what's broken, who's on it, and how long it's taking across your PEAK-monitored sites.

## Pre-requisites

This skill requires the **PEAK MCP connector** to be installed and authenticated in Claude.

- MCP URL: `https://api.cimenviro.com/mcp`
- Auth: OAuth 2.0

In Claude, go to **Settings > Connectors > Add custom connector**, paste the URL above, and complete the OAuth sign-in with your PEAK account.

## Install

1. Click **Code > Download ZIP** from the Github repo
2. In Claude, go to **Customize > Skills > Create Skill > Upload a skill**
3. Upload the ZIP file as a skill

## Usage

From any Claude Chat session, run `/peak-ops` and tell it which sites or portfolio to look at. The skill returns markdown tables of P1-2 alerts in fault by site, with optional drilldowns by equipment type, equipment name, or full alert list.

For example:

- `/peak-ops what requires attention across the following sites: Building 12, 123 Main St, Riverside Plaza`
- `/peak-ops what requires attention across [customer name] sites`

After the summary table renders, ask follow-ups like *"drill into Building 12 by equipment type"* or *"show me the full alert list for 123 Main St"*.
