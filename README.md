# Xylo plugins

Official Codex plugin marketplace for [Xylo](https://xylomcp.com), the MCP server that gives AI agents access to Meta Ads, Google Ads, TikTok Ads, X Ads, and Klaviyo.

## Install

Add the marketplace once:

```bash
codex plugin marketplace add XyloMCP/plugins
```

Then open **Plugins** in the Codex desktop app, or type `/plugins` in Codex CLI, and install **Xylo**. Authenticate through the browser when prompted. Xylo uses OAuth, so no API key is required.

Start a new task after installation so Codex loads both the Xylo MCP tools and the bundled Xylo Ads skill.

## Update

Backend improvements arrive automatically through Xylo's remote MCP server. When Xylo announces a plugin update, open **Settings → Plugins → Xylo**, select **Refresh**, and start a new task. CLI users can run:

```bash
codex plugin marketplace upgrade xylo
```

## Included

- Xylo's production OAuth MCP connection at `https://xylomcp.com/api/mcp`
- The Xylo Ads skill for Meta, Google, TikTok, and X campaign workflows plus Klaviyo
- Xylo presentation assets and marketplace metadata

## Links

- [Documentation](https://xylomcp.com/docs)
- [Privacy policy](https://xylomcp.com/privacy)
- [Terms of service](https://xylomcp.com/terms)
- [Support](mailto:support@xylomcp.com)
