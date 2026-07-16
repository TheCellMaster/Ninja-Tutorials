# OGame Ninja — Tutorials & Advanced Reference (Unofficial)

Practical guides and a **machine-readable reference** for advanced
[OGame Ninja](https://www.ogame.ninja) bot users — built so you can **feed it to an AI** and have it help
you write scripts and panel-automation code.

> ⚠️ Unofficial project. Not affiliated with Gameforge or OGame Ninja. For educational use.

---

## 🤖 Feed this to your AI (LLM)

This repo ships an **anonymized, LLM-ready reference bundle** in [`ninja-reference/`](ninja-reference/).
Its whole purpose is to give an AI assistant **awareness of what OGame Ninja can do** — its control panel
(HTTP API + database), browser automation, module behavior and error handling — so the AI can help you
build real automation projects instead of guessing.

Two ways to load it:

| Goal | Use this |
|---|---|
| **Point an AI / RAG pipeline at the docs** | [`ninja-reference/llms.txt`](ninja-reference/llms.txt) — a manifest that routes the model to the right file |
| **Paste or upload everything in one shot** | [`ninja-reference/ninja-reference-public-full.md`](ninja-reference/ninja-reference-public-full.md) — every doc concatenated into a single file |

> **Example prompt:** *"Read the attached OGame Ninja reference and help me write a Python + Playwright
> script that creates a bot and uploads an Anko script through the panel API."*

---

## 📚 What's inside `ninja-reference/`

Focused on **driving and operating the bot programmatically**. Writing `.ank` scripts is intentionally
left to the [official documentation](https://www.ogame.ninja/doc), so this bundle does not duplicate it.

| Layer | Folder | Contents |
|---|---|---|
| **Control plane** | [`01-control-plane/`](ninja-reference/01-control-plane/) | Panel HTTP API + SQLite schema; panel URLs + CSS selectors (Playwright) |
| **Operation** | [`02-operation/`](ninja-reference/02-operation/) | Module behavior (Brain / Scanner / Hunter / Defender / Farmer / …) + error & troubleshooting catalog |

Every file starts with YAML frontmatter (`title`, `category`, `layer`, `tags`, `summary`) for clean
RAG chunking and retrieval.

---

## 🔒 Anonymized

All example values are **fictional placeholders** (`ServerA`, `sXXX-en`, `EXAMPLE_*`,
`user@example.com`, RFC 5737 documentation IPs). Nothing here is real configuration — treat every value
as an example.

---

## 📖 More

- Step-by-step guides: **[Wiki](https://github.com/TheCellMaster/Ninja-Tutorials/wiki)**
- Official OGame Ninja docs (scripting, setup): <https://www.ogame.ninja/doc>
