# OGame Ninja — Advanced Control & Operation Reference

**Anonymized** reference documentation for advanced OGame NinjaBot users, organized to be consumed by AIs
(RAG/LLM) or read by humans. It focuses on **driving and operating the bot programmatically**:

- **Controlling the bot's own panel** via code (HTTP API + SQLite database), e.g. Python/Playwright.
- **Operating** a fleet of bots: module behavior and error handling.

> ℹ️ **Not here:** writing Anko `.ank` scripts. That in-bot API is fully covered by the official docs
> (<https://www.ogame.ninja/doc>), so this bundle deliberately does not duplicate it.

> 🤖 **To feed an AI:** start with [`llms.txt`](llms.txt) (manifest), or send the single file
> [`ninja-reference-public-full.md`](ninja-reference-public-full.md) (all docs concatenated).

## How it's organized

| Layer | Folder | What it is |
|---|---|---|
| **Control plane** | [`01-control-plane/`](01-control-plane/) | Drive the panel via code (HTTP + DB + selectors) |
| **Operation** | [`02-operation/`](02-operation/) | Module behavior + error catalog |

Each `.md` file has **YAML frontmatter** (`title`, `category`, `layer`, `tags`, `summary`) to make
chunking/retrieval by an AI easier.

## Index

| Doc | Layer | Description |
|---|---|---|
| [panel-http-api](01-control-plane/panel-http-api.md) | control-plane | Panel HTTP endpoints + database schema |
| [panel-pages-and-selectors](01-control-plane/panel-pages-and-selectors.md) | control-plane | URLs + HTML/CSS selectors (Playwright) |
| [bot-modules](02-operation/bot-modules.md) | operation | Brain/Scanner/Hunter/Defender/Farmer/… |
| [errors-and-troubleshooting](02-operation/errors-and-troubleshooting.md) | operation | Known errors + recommended reaction |

## Scripting (external)

Writing `.ank` scripts is out of scope for this bundle — use the official reference:

- Docs: <https://www.ogame.ninja/doc>
- Examples: <https://github.com/ogame-ninja>

## Notice

**All data has been anonymized** with fictional placeholders (`ServerA`, `sXXX-en`, `EXAMPLE_*`,
`user@example.com`, RFC 5737 documentation IPs). **No value is real** — treat everything as an example.
It is not usable configuration.
