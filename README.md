# Artifen plugin

[Artifen](http://localhost:3000) hosts whatever your AI builds, from one HTML page to a full-stack app with a database, sign-in, files, email, AI, and payments, each at its own address. This plugin connects your agent to Artifen and adds the `artifen` skill, which teaches it when to put something online there, how to build and change a site without losing anyone's work, how to publish, and how to recover from errors.

## Install

In a terminal:

```bash
# Claude Code
claude plugin marketplace add artifen-app/Artifen-Plugin#dev
claude plugin install artifen@artifen

# Codex
codex plugin marketplace add artifen-app/Artifen-Plugin --ref dev
codex plugin add artifen@artifen

# Gemini CLI
gemini extensions install https://github.com/artifen-app/Artifen-Plugin --ref dev

# Antigravity CLI
git clone -b dev https://github.com/artifen-app/Artifen-Plugin artifen-plugin && agy plugin install ./artifen-plugin
```

Inside a Claude Code session, `/plugin marketplace add artifen-app/Artifen-Plugin#dev` and `/plugin install artifen@artifen` do the same.

- **Cursor**: clone this repository into `~/.cursor/plugins/local/artifen`, then run Developer: Reload Window.
- **Claude and ChatGPT on the web**, and any other app that connects MCP servers: add `http://localhost:3003/mcp` as a remote server, as [Connect your AI](http://localhost:3000/connect) shows step by step. This adds the tools without the skill.

## Sign in

The first time your agent uses Artifen, your browser opens to sign in to Artifen and allow the app. If you have no Artifen account yet, signing in makes one. To start it yourself:

- **Claude Code**: run `/mcp`, choose `plugin:artifen:artifen`, then Authenticate.
- **Codex**: run `codex mcp login artifen`.
- **Gemini CLI**: run `/mcp auth artifen`.
- **Antigravity CLI**: your browser opens to sign in when Artifen first connects.

The connection reaches every workspace you belong to and every site shared with you, with your role in each. You can disconnect an app at any time under Connected agents in the Artifen app's settings.

## Update

```bash
# Claude Code
claude plugin marketplace update artifen
claude plugin update artifen@artifen

# Codex
codex plugin marketplace upgrade artifen
codex plugin add artifen@artifen

# Gemini CLI
gemini extensions update artifen

# Antigravity CLI
git -C artifen-plugin pull --ff-only && agy plugin uninstall artifen && agy plugin install ./artifen-plugin
```

Start a new session afterwards so your agent picks up the new skill and tools.

## What your agent can do

Create sites, change their files, check each build, publish a site's preview, list and restore earlier versions, read build and server logs, query a site's data, store secrets, upload files, and read and answer comments. Sites in Trash can be restored for 30 days. Who can open a site, custom domains, billing, and deleting a site for good are yours to change in the [Artifen app](http://localhost:3001).

## Help

- Support: [artifen.app/contact](http://localhost:3000/contact), or contact@artifen.app
- [Privacy Policy](http://localhost:3000/privacy)
- [Terms of Service](http://localhost:3000/terms)

Licensed under the [Apache License 2.0](LICENSE).
