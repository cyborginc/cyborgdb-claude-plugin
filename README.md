# CyborgDB Claude Code Plugin

A [Claude Code](https://docs.claude.com/en/docs/claude-code) plugin marketplace that helps you set up and use [CyborgDB](https://cyborg.co) — the confidential vector database — directly from your editor. Claude detects the CyborgDB version you're on, pulls the matching docs, and scaffolds a working integration for you.

## Install

Inside Claude Code, add the marketplace and install the plugin:

```
/plugin marketplace add cyborginc/cyborgdb-claude-plugin
/plugin install cyborgdb-skills@cyborgdb-public
```

That's it. The skills below activate automatically when your request matches — or you can invoke them by name.

## What you get

The `cyborgdb-skills` plugin bundles three skills:

| Skill | Use it when you want to… |
|-------|--------------------------|
| **cyborgdb-core** | Embed the in-process CyborgDB engine directly in your Python app (no server). |
| **cyborgdb-service** | Start and run the standalone CyborgDB HTTP server, then verify it's healthy. |
| **cyborgdb-sdks** | Connect an app to a running `cyborgdb-service` from Python, JavaScript/TypeScript, or Go. |

Each skill detects the version you're actually on, points at the matching `v0.X.x` docs on [docs.cyborg.co](https://docs.cyborg.co), and writes a minimal working integration following that version's quickstart.

## Example prompts

Once installed, just ask Claude naturally:

- "Set up cyborgdb-core in my Python project"
- "Start cyborgdb-service with Docker and check it's healthy"
- "Connect my app to cyborgdb-service using the TypeScript SDK"

## Which one do I need?

- **In-process, Python only, no server** → `cyborgdb-core`
- **Running a server** → `cyborgdb-service`, then `cyborgdb-sdks` to call it from your app
- **Any language other than Python** → `cyborgdb-service` + `cyborgdb-sdks`

## Links

- Docs: [docs.cyborg.co](https://docs.cyborg.co)
- Website: [cyborg.co](https://cyborg.co)

## Maintained by

The Cyborg Team. Issues and contributions welcome.
