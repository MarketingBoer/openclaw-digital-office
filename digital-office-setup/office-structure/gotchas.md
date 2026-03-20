# Gotchas — office-structure

Real pitfalls encountered in production. Read before configuring a new office.

---

## Gotcha 1: bootstrapTotalMaxChars=150000 silently truncates

**What happens:** If the total size of all workspace files exceeds 150,000 characters, OpenClaw silently truncates the content that doesn't fit. The agent starts without error but with an incomplete context window.

**Symptom:** Agent behaves inconsistently — sometimes follows instructions from AGENTS.md, sometimes doesn't. No error message. Looks like a model hallucination problem.

**Fix:** Keep workspace files lean. Use memory/ for details. Reserve SOUL.md + AGENTS.md as your most important files — front-load them. Check total with:
```
wc -m workspace-{name}/*.md
```
Stay well under 150,000 total. The 20K per-file limit applies too — files over 20K are silently capped.

---

## Gotcha 2: BOOTSTRAP.md re-executes if not deleted

**What happens:** BOOTSTRAP.md is designed as a one-time first-run ritual. But if you forget to delete it after the first run, it re-executes on every gateway restart. The agent performs setup steps (creating directories, sending welcome messages, configuring integrations) repeatedly.

**Symptom:** Agent sends duplicate welcome messages, re-initializes already-configured services, overwrites memory from previous sessions.

**Fix:** After the first-run is confirmed complete, delete BOOTSTRAP.md:
```
rm workspace-{name}/BOOTSTRAP.md
```
This is distinct from BOOT.md, which IS meant to run on every restart.

---

## Gotcha 3: dmScope default is NOT per-channel-peer

**What happens:** The default `dmScope` is NOT `per-channel-peer`. Without explicit configuration, all users messaging the same bot share a single session. User A's conversation history bleeds into User B's session.

**Symptom:** Agent references context from other users' conversations. Privacy issue in multi-user inboxes. Especially dangerous on WhatsApp where multiple people may DM the same number.

**Fix:** Always set explicitly:
```json
{ "sessions": { "dmScope": "per-channel-peer" } }
```
Set this in `agents.defaults.sessions` to apply globally to all agents.

---

## Gotcha 4: archiveAfterMinutes=60 can lose sub-agent context

**What happens:** The default `archiveAfterMinutes=60` archives sub-agents after 60 minutes of inactivity. For long-running research tasks, the sub-agent gets archived mid-task. The parent agent loses the sub-agent context and may retry, spawning a new sub-agent and doubling cost.

**Symptom:** Research tasks take twice as long as expected. Token usage spikes. Sub-agents restart from scratch instead of resuming.

**Fix:** Increase `archiveAfterMinutes` for research workspaces:
```json
{ "subagents": { "archiveAfterMinutes": 240 } }
```
For fast classifiers, decrease it to 15 to free resources quickly.

---

## Gotcha 5: TOOLS.md does not restrict tools — it only gives guidance

**What happens:** Developers often write TOOLS.md with instructions like "do not use exec tools" expecting it to prevent tool use. TOOLS.md is a guidance document that influences agent behavior — it does NOT enforce tool availability.

**Symptom:** Agent "decides" to use exec tools despite TOOLS.md instructions, especially in edge cases or under prompt injection.

**Fix:** For actual tool restriction, use the `tools.deny` array in `openclaw.json` agent config. TOOLS.md is reinforcement only. See security/ for the full deny-list pattern for read-only agents.
