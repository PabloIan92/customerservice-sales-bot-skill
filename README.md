# lemmonbot-maintenance skill

A [Claude Code](https://claude.com/claude-code) skill distilling the methodology and hard-won gotchas from building and maintaining a WhatsApp/Telegram support+sales chatbot on **n8n + Chatwoot**, both self-hosted.

This is **not** a copy of any specific bot's workflow, and contains **no credentials, tokens, server addresses, or business data**. It's a reusable reference for the *patterns* that came up repeatedly: safely editing a large stateful Code node through a REST API, avoiding a nasty escape-sequence corruption bug across multi-layer tool pipelines, patching a self-hosted app's frontend without breaking its build, and diagnosing "it's down" reports methodically.

## Install

Drop the `.claude/skills/lemmonbot-maintenance/` folder into a project (or your global `~/.claude/skills/`), and Claude Code will pick it up automatically.

## Contents

- `.claude/skills/lemmonbot-maintenance/SKILL.md` — the skill itself.
