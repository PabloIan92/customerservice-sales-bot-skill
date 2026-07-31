# customerservice-sales-bot skill

A [Claude Code](https://claude.com/claude-code) skill for building and maintaining a **free, self-hosted customer-service + sales chatbot** on **n8n + Chatwoot** — no per-seat SaaS licensing, just a small VPS.

Covers the whole lifecycle: architecture, how to wire the bot up to external APIs (CRM/billing lookups, calendar booking, team-chat alerts), the conversation state-machine design patterns, safely editing a large automation workflow through its REST API without corrupting it, patching a self-hosted app's frontend, and diagnosing "it's down" reports.

Contains **no credentials, tokens, server addresses, or business-specific data** — it's a reusable reference for the *patterns*, not a copy of any one bot.

## Install

Drop the `.claude/skills/customerservice-sales-bot/` folder into a project (or your global `~/.claude/skills/`), and Claude Code will pick it up automatically.

## Contents

- `.claude/skills/customerservice-sales-bot/SKILL.md` — the skill itself.
