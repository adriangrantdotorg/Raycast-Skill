# 🔍✨ Raycast Agentic Skill

![Raycast Agentic Skill Banner](banner.png)

> A free skill file that teaches your AI coding assistant how to build Raycast commands and extensions the right way, the first time.

[![License](https://img.shields.io/badge/License-Apache%202.0-blue.svg)](LICENSE.txt)
[![Raycast API](https://img.shields.io/badge/Raycast%20API-2.5.0-FF6363.svg)](https://developers.raycast.com/)
[![Updated](https://img.shields.io/badge/Updated-09--25--26-orange.svg)](https://github.com/adriangrantdotorg/Raycast-Skill/commits/main)
[![Agent Skills](https://img.shields.io/badge/Agent%20Skills-Standard-8A2BE2.svg)](https://github.com/anthropics/skills)
[![PRs Welcome](https://img.shields.io/badge/PRs-welcome-brightgreen.svg)](https://github.com/adriangrantdotorg/Raycast-Skill/pulls)

---

## ⬇️ Why Install?

- ⏱️ **Less time per command** — it gets things right on the first try
- 💸 **$0 added cost** — it rides on the AI you already pay for
- 📅 **Current with Raycast API 2.5.0** — not the AI's year-old memory

| | 😩 Without this Skill | 😌 With this Skill |
| --- | :---: | :---: |
| 🔁 Tries until it works | 🧪 3–5 | **1** |
| 📄 Code for a simple hotkey | 🧪 ~200 lines | **~10 lines** |
| 🛠️ Build steps | `npm install` + build | **none** |
| 🪙 AI usage burned on retries | 🧪 ~4× | **1×** |

<sub>🧪 estimate</sub>

---

## ✨ Features

Before writing code, the AI picks the lightest option that works: a built-in Quicklink or Snippet, a one-file Script Command, or a full Extension.

![The same request with and without the skill: without it, the AI guesses 212 lines, builds, and fails three times before it works on try 4; with it, the AI checks the rules, selects a 9-line Script Command, and it works on try 1, leaving the rest as time you get back](docs/media/with-vs-without-skill.svg)

- 📜 **Script Commands that don't hang** — every `@raycast.*` setting right, including traps like a hidden SSH password prompt
- 🎨 **Extensions that feel native** — keyboard-driven lists, self-closing forms, menu bar items, background refresh
- ⚡ **Instant, flicker-free data** — the right `@raycast/utils` hook for each job
- 🔐 **Secure tokens and sign-in** — encrypted preferences and built-in OAuth for GitHub, Linear and Slack
- 🤖 **Raycast AI tools** — tools Raycast AI can call for you ("@contacts find…")
- 🩺 **Debugging that finds the real error** — like a failed rebuild that silently keeps the old code running

| | **This skill** | [GitHub Copilot](https://github.com/features/copilot/plans) | [Cursor](https://cursor.com/pricing) | [Context7](https://context7.com/plans) |
| --- | :---: | :---: | :---: | :---: |
| 💰 Price | **Free** | $0–10/mo | $0–20/mo | $0–10/mo |
| 🧭 Built-in Raycast rules | ✅ | ❌ | ❌ | ❌ |
| 📚 Current Raycast API facts | ✅ | ⚠️ | ⚠️ | ✅ |
| 🔌 Works in any AI coding tool | ✅ | ❌ | ❌ | ✅ |
| 🔓 Open source | ✅ | ❌ | ❌ | ⚠️ |
| ✈️ Works offline | ✅ | ❌ | ❌ | ❌ |

<sub>✅ yes · ⚠️ partly · ❌ no · checked 09-25-26</sub>

---

## 🚀 Installation

Needs **[Raycast](https://www.raycast.com/)** and an AI assistant that supports [Agent Skills](https://github.com/anthropics/skills). Full Extensions also need **[Node.js](https://nodejs.org/)** 22.22+.

```bash
git clone https://github.com/adriangrantdotorg/Raycast-Skill.git /tmp/Raycast-Skill
cp -R /tmp/Raycast-Skill/skills/raycast ~/.claude/skills/
```

Swap `~/.claude/skills/` for your tool's folder:

| **Platform** | **Skills folder** |
| --- | --- |
| **Claude Code** · **Raycast AI** | `~/.claude/skills/` |
| **Cursor** | `.cursor/skills/` in your project |
| **OpenAI Codex** | `~/.codex/skills/` |
| **Gemini CLI** | `~/.gemini/skills/` |
| **Google Antigravity** | `.agent/skills/` in your project |
| **OpenCode** | `~/.config/opencode/skills/` |

Keep the folder named `raycast`; some tools skip a skill whose folder name doesn't match.

---

## 💡 Usage

Describe what you want; the skill kicks in on its own.

| You say | The skill makes |
| --- | --- |
| "Make a Raycast command that wakes my Mac Mini over the network." | A **Script Command** in silent mode that shows a clear error toast if it fails |
| "Build a Raycast extension that adds a task to my Notion database, with a tag picker." | An **Extension** with the token stored securely, cached tags, and a form that closes when saved |
| "Create a command that runs my Keyboard Maestro macro." | A command that passes the macro ID safely to `runAppleScript`, marked macOS-only |

---

<div align="center">
  <sub>Built with ❤️ for the Raycast community & everyone teaching their AI new tricks ✌🏾</sub>
</div>
