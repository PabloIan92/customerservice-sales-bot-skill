---
name: lemmonbot-maintenance
description: Methodology and gotchas for maintaining LemmonBot, a WhatsApp/Telegram support+sales bot built on n8n + Chatwoot for an Argentine ISP. Use when editing the bot's n8n workflow, patching self-hosted Chatwoot, or debugging the conversation state machine.
---

# LemmonBot Maintenance

This skill captures the **methodology and hard-won gotchas** from building and maintaining LemmonBot, a customer-support and sales chatbot for an ISP. It does **not** contain any credentials, tokens, server IPs, or business-specific data — those live in the operator's own notes/password manager. Adapt the concrete node names, states, and business rules described here to whatever bot you're actually working on; what matters is the *pattern*.

## Architecture at a glance

- **n8n** (self-hosted) orchestrates the bot. A single giant **Code node** (referred to here as `clasificador`) is the heart of the system: it's a state machine keyed on a `conversacion_estado` value stored in the chat platform's per-conversation custom attributes.
- **Chatwoot** (self-hosted) is the chat platform/inbox. It receives messages via webhook (Telegram/WhatsApp/Email channels), and the n8n workflow posts replies back via its REST API.
- External integrations typically include: a CRM/billing API for customer lookup, a calendar API (e.g. Google Calendar) for appointment booking, and a team chat webhook (e.g. Slack) for internal alerts.
- The whole thing runs on a small VM (often 1 vCPU) — this matters for build/compile steps later.

## The state-machine Code node

- Each incoming message re-runs the entire Code node from scratch. It reads the previous `conversacion_estado`, looks at the message content, and decides a new state + a reply + optional side-effect flags (an `accion` value that downstream nodes act on, like "send a payment link" or "reboot a device").
- **Critical gotcha — authoritative last-write.** If there's a downstream node that does a final "save custom attributes" HTTP call using an expression like `Object.assign({}, preTurnSnapshot, extraOut, {conversacion_estado: nuevoEstado})`, then **any direct API call made *inside* the Code node itself to update custom attributes gets silently clobbered** by that later authoritative write — because it only knows about the pre-turn snapshot plus whatever you routed through the `extraOut`-equivalent object. Any flag that must persist has to flow through that shared "extra output" object, never only via a direct side-channel API call from inside the Code node.
- **Critical gotcha — substring vs. word-boundary matching.** If there's a shared helper like `match(text, ...keywords)` that does plain substring search (`text.indexOf(keyword) !== -1`), never use short, common tokens through it — e.g. "no" is a substring of "bueno", "si" is a substring of "servicio". For short ambiguous words, use exact equality (`text === "no"`) instead, and reserve substring `match()` for longer, more specific phrases.
- **Order matters for accept/reject detection.** When a state has to distinguish "customer accepted" vs "customer rejected" vs "ambiguous, ask again", check rejection keywords *before* acceptance keywords if a compound reply could contain both (e.g. "sí, pero prefiero cancelar igual").
- **Give the AI concrete facts, not vibes.** When a state hands off to an LLM step, always inject the concrete data it needs (customer's real name, exact prices, exact business rules) into the per-turn instruction rather than trusting the model's memory of the conversation. Small/cheap models especially will paraphrase, invent, or truncate names if not given the literal string to use verbatim, and may fall back to reproducing a stale template from the conversation history when given no concrete instruction at all.
- **A narrowly-scoped agent beats a bolted-on instruction.** If a general-purpose AI step with many tools and a long system prompt won't reliably follow a new nuanced instruction, it's often more reliable to build a small, dedicated AI step with a short, focused system prompt for that one job, rather than continuing to patch the shared prompt.
- **"Transient" states must actually reset.** If a state value like `ai_chat` is meant to mean "let the AI answer this one message" rather than a real durable state, make sure whatever persists state after the AI responds does *not* write that transient value back as the new `conversacion_estado` — otherwise the conversation gets stuck in a state with no matching branch, forever falling through to the generic fallback on every subsequent message.
- **Contact-level vs. conversation-level data.** If customer identity and per-conversation state live in different records (e.g. Chatwoot's "contact" vs. "conversation"), any merge logic that copies contact-level facts into the conversation should not be gated on unrelated fields being empty — a common bug is "only backfill contact data if the conversation doesn't already have a customer ID yet", which silently breaks for already-identified returning customers.
- **3-strikes-then-escalate.** For any free-text state where the bot might genuinely not understand, track a small per-conversation retry counter and escalate to a human after a handful of failures, rather than looping forever or guessing indefinitely. Reset the counter whenever the customer is successfully understood again.

## Safe workflow-editing loop (via the workflow platform's REST API)

When editing a large Code node's source through a script (rather than an IDE):

1. **Fetch fresh** — always re-GET the current workflow JSON right before editing; don't reuse a stale copy from earlier in the session.
2. **Locate by exact substring, never by retyping.** Search for a unique anchor string and verify `code.count(anchor) == 1` (or exactly the expected count) *before* replacing. If a `.replace()` call "succeeds" against the wrong number of matches, you'll silently corrupt something else.
3. **Extract, don't retype, anything with escape sequences or non-ASCII characters.** Large auto-generated or hand-edited source files often mix encoding styles inconsistently — some sections store emoji/accents as raw UTF-8 bytes, others store them as literal `\uXXXX` escape text. If you retype what you *think* the bytes are, you will very likely get it wrong in a way that's invisible until the deploy fails. Instead, slice the exact existing bytes out of the fetched source (`code[start:end]`) and only mutate the plain-ASCII portions you actually need to change.
4. **Constructing new escape sequences (like `\n`) programmatically, not literally.** When generating brand-new string content that needs an embedded newline or backslash, build it with `chr(92) + "n"` (or your language's equivalent) rather than typing the literal two-character escape sequence `\n` in your script. Multi-layer tool/shell pipelines (heredocs, SSH, nested quoting) can silently transform a literal backslash-n into a real newline character before it reaches the target file, corrupting a string literal into invalid syntax. This class of bug is sneaky because the corruption is invisible in your own source and only shows up as a syntax error downstream.
5. **Deploy ritual**: PUT the updated workflow body → deactivate → activate (many platforms don't hot-reload from PUT alone) → re-GET to confirm the change landed.
6. **Always syntax-check before calling it done.** Pull the deployed code back out and run it through your language's syntax checker wrapped in a bare function shell (e.g. Node's `node --check` on the code wrapped in an async IIFE). This is the single highest-leverage safety net in this whole loop — it catches brace-mismatches and encoding corruption *before* they reach production, where a broken Code node can mean the bot stops responding to every single message.
7. **When you *do* introduce a syntax error anyway** (it happens), don't just re-guess the fix — programmatically count brace/paren depth through the suspect region to pinpoint the exact byte offset where it goes negative or fails to return to zero. Trying to eyeball-diff a large generated block wastes far more time than a ten-line depth counter.

## Testing changes

- Keep one dedicated test conversation (a real conversation in the chat platform, linked to a real or sandbox backend record) that you reset between test rounds via a direct API call to clear/reset its custom attributes to a known starting state.
- After any deploy, actually drive a test conversation through the new logic rather than only trusting the syntax check — syntax-valid code can still have wrong business logic, wrong state names, or wrong field names.
- When a live test reveals unexpected bot behavior, don't guess — pull the actual conversation history and the actual current custom-attribute values via the API and reconstruct exactly which branch fired, especially before concluding "the AI is being unreliable" (it's often a deterministic routing bug instead).

## Patching a self-hosted web app's frontend (e.g. Chatwoot)

If you need to add a UI feature to a self-hosted open-source app rather than a small automation script:

1. **Look for existing infrastructure first.** Before adding a whole new feature, check whether the underlying capability already exists in the app (e.g. a generic "bulk action" pipeline, a "delete single item" endpoint) — extending existing hooks is far lower-risk than inventing a parallel mechanism.
2. **Match the existing component style exactly**, not just its function. A common mistake: copying a sibling component's *behavior* but not its *visual pattern* (e.g. adding a labeled button next to a row of icon-only buttons) — this silently breaks the layout of a space-constrained container. Actually look at a sibling component's template before assuming what "looks right."
3. **Verify visually, not just functionally**, for anything UI-related. A working button that renders in the wrong place, wraps oddly, or overflows its container is still a bug. Take a real screenshot after the change; don't infer correctness from the code alone. If a headless browser session is already open from a previous task, it may be stale — check for and clean up orphaned browser processes bound to the same automation profile before starting a fresh session.
4. **Rebuild environment gotchas are easy to get backwards.** A systemd/service unit's declared `PATH` environment variable is not gospel — check what the *actual* running process resolves (e.g. via `/proc/<pid>/environ`) versus what a login shell resolves, since version managers (rbenv/rvm/nvm/etc.) are often set up via shell-login hooks that a bare non-login shell won't run. If a build step needs the same toolchain the service uses, invoke it through a login shell rather than trusting the unit file's literal env vars.
5. **Frontend build memory limits.** On a small VM, a full production frontend build (bundling thousands of modules) can exceed the JS runtime's default heap size and crash with an out-of-memory error that has nothing to do with actual available system RAM. Explicitly raise the heap limit for the build command rather than assuming the server needs more physical memory.
6. **Track custom patches as reapplicable diffs.** Any change made directly to a third-party app's source will be overwritten or conflict on the next update/upgrade. Save each patch as its own diff file (both on the server and in your own notes), with the exact rebuild/restart procedure written down next to it, so "reapply the customization" is a five-minute mechanical task instead of a rediscovery project.

## Diagnosing "it's down" reports

When told something is "down" or "broken":

- **Check the actual service first**, locally on the server (process status, direct localhost request), before assuming the public-facing issue and the backend issue are the same thing. A public URL/tunnel/proxy can be down while the underlying app is perfectly healthy, and vice versa.
- **Read the actual request logs around the reported time** rather than guessing. A slow or hung request (e.g. one that times out trying to validate an external connection) can look to a user exactly like "the whole app crashed," when really one specific action is just slow and blocking their perception of the UI.
- **A misconfigured field (wrong port, typo) is a much more common root cause than an actual outage** — check the literal parameters of the action that was being performed when things "broke" before escalating to infrastructure-level debugging.
- If the report turns out to be a tunneling/mesh-networking layer (e.g. a reverse proxy service) having silently lost its connection to its coordination servers while the local app stays healthy: check that layer's own connectivity/status command, not just "is the port open."

## General working style for this kind of bot

- Business-logic corrections often arrive as quick corrections to something just shipped ("that's backwards", "don't say X there") — treat these as direct, authoritative specification, verify the current code reflects the correction precisely, and redeploy promptly rather than batching them.
- When a user distinguishes between two similar-looking flows (e.g. "the closing message for an existing customer moving isn't the closing message for a brand-new sale"), that's a signal there are two logically distinct paths sharing code that need to diverge — find the shared code and branch it, rather than editing the one path in a way that silently changes the other too.
- Before implementing a "show me X" request for a UI panel, check what data is actually queryable/stored right now versus what would need new instrumentation — don't silently under-deliver by omitting a requested field, and don't silently fabricate a field that doesn't exist. Say plainly which parts are ready now and which need a new data source.
