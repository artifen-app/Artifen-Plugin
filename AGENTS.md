# Artifen Plugin

This repository is the Artifen agent plugin: the `artifen` skill and the connection to Artifen's MCP server, packaged for each agent host. Artifen publishes it from its own source, and each publish replaces these files, so a change made here directly does not last. To suggest a change, write to contact@artifen.app.

## Layout

| Path | What it is |
| --- | --- |
| `plugin.json` | The plugin's name, version, description, and links, in Agent Plugins format, read by Codex and Cursor |
| `mcp.json` | The MCP connection in Agent Plugins format |
| `.claude-plugin/plugin.json`, `.mcp.json`, `.claude-plugin/marketplace.json` | Claude Code's descriptors |
| `.agents/plugins/marketplace.json` | Codex's marketplace entry |
| `gemini-extension.json` | Gemini CLI's descriptor |
| `skills/artifen/` | The skill |
| `assets/icon.png` | The icon hosts show |

Every descriptor is written from `plugin.json`, so they always carry the same name, version, and server address. Branch `main` connects to Artifen, and `preview` to its staging copy.

To try the plugin from a clone, load the folder directly, such as `claude --plugin-dir <clone>`. `claude plugin validate --strict <clone>` checks it.
