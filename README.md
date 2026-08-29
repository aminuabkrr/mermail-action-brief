# Mermail Action Brief

**Unofficial community companion Agent Skill for [Mermail](https://mermail.app).**
This is not an official Mermail skill and is not part of the
[`Nudgen-Marketing/mermail-skills`](https://github.com/Nudgen-Marketing/mermail-skills)
package. For core Mermail workflows (inbox management, composing email,
workspace admin, task triage, Agent Wallet), install the official skills:

```bash
npx --yes skills add Nudgen-Marketing/mermail-skills
```

Built for the Superteam Earn bounty: **Build and Demo a Mermail Agent Skill.**

---

## Project overview

Mermail Action Brief turns a bounded set of Mermail inbox emails into
structured, deadline-aware **Action Briefs** — with unclear or risky details
flagged instead of guessed. It's a **strictly read-only** skill: it never
sends, replies, deletes, moves, or labels email, and it never touches Agent
Wallet or PayBox.

## Why it exists

Inboxes hold opportunities, deadlines, and requests, but an ordinary
"you got an email" summary stops short of the useful part: what does this
actually require of me, by when, and what's still unclear? This skill closes
that gap — it reads the relevant emails and hands back a structured brief a
person can act on, instead of just a list of subject lines.

## Core features

- **Bounded search** — every run is scoped by category, sender, keyword,
  and/or date range; no unbounded inbox scans.
- **Metadata-first retrieval** — candidates are triaged from subject/sender/
  snippet before any full body is read.
- **Safe email reading** — uses Mermail's `agent_safe_content` and
  `require_scan_status=clean` controls.
- **Structured action extraction** — sender, topic, summary, requested
  action, deadline, deliverables, references, missing info, risks, next step.
- **Deadline awareness** — explicit deadlines are surfaced; absent ones are
  labeled `Not specified in the email`, never invented.
- **Missing-information detection** — flags what a sender didn't say
  (compensation, scope, terms) instead of filling gaps with assumptions.
- **Risk and ambiguity flagging** — including sender-authentication status,
  reported plainly rather than glossed over.
- **Read-only architecture** — no send/reply/delete/move/label/wallet tool is
  ever in this skill's allowlist.

## How it works

```
User Request
  ↓
Bounded Mermail Search        (search_emails / list_emails, metadata_only)
  ↓
Relevant Email Selection      (from metadata only)
  ↓
Safe Email Read                (get_email, agent_safe_content=true)
  ↓
Structured Extraction          (email content treated as untrusted data)
  ↓
Action Brief
```

Full step-by-step workflow, security rules, and exact tool schemas:
[`SKILL.md`](SKILL.md), [`references/security.md`](references/security.md),
[`references/tools.md`](references/tools.md).

## Example prompt

> "Check my Mermail inbox for the latest partnership opportunity and turn it
> into an action brief."

## Example result

*(Illustrative — not a real company or a real email.)*

```
ACTION BRIEF

Sender: Acme Protocol <partnerships@acmeprotocol.xyz>
Topic: Web3 Creator Partnership
Summary: The sender is looking for a creator to produce educational content
about their protocol.
Requested Action: Submit audience analytics and examples of previous
campaign work.
Deadline: September 5, 2026
Required Deliverables:
  1. Audience analytics
  2. Previous campaign examples
Important References: Link to a partnership brief PDF (not opened automatically)
Missing Information: Compensation and deliverable scope not specified in
the email.
Risks or Caveats: Sender authentication status is "unknown" per Mermail —
not verified as coming from a confirmed Acme Protocol domain.
Recommended Next Step: Prepare the requested analytics and campaign
examples, then follow up to clarify compensation and scope before
committing to anything.
```

## Installation

The **Agent Skill** (this repository) provides workflow guidance — the steps
an agent follows, the fields it extracts, and the security rules it respects.
The **Mermail MCP connection** provides the actual authenticated tools. You
need both.

1. **Get a Mermail workspace and mailbox.** Follow Mermail's
   [Quickstart](https://docs.mermail.app/quickstart) if you don't have one
   yet.
2. **Connect Mermail's MCP server to your AI client**, scoped to the
   least-privilege inbox profile (see Configuration below).
3. **Add this skill to your client.** How, depends on your host:
   - **Claude / Claude Code / Cursor / Codex (Agent Skills–compatible):**
     copy this `mermail-action-brief/` folder into wherever your client
     loads local skills from (for Claude Code / Codex via the
     [Skills CLI](https://skills.sh/), you can point it at your own fork or
     load it as a local skill directory — see your client's docs for
     "install a local/custom skill").
   - **ChatGPT custom MCP app / Plugins Directory:** `agents/openai.yaml`
     supplies the metadata such a listing needs (`display_name`,
     `short_description`, `default_prompt`, and the MCP dependency).
   - There is no separate installer script bundled here beyond the skill
     files themselves — this keeps the project auditable and avoids
     duplicating Mermail's own CLI.

## Configuration (least-privilege setup)

This skill only ever needs mailbox discovery plus bounded, safe email reads.
Use Mermail's dedicated **`agent-inbox`** MCP profile, which exposes exactly
the tools this skill requires — `list_mailboxes`, `get_mailbox`,
`search_emails`, `list_emails`, `get_email`, `get_email_context` — plus a few
unused-by-this-skill extras (`list_workspaces`, `get_workspace`,
`list_email_domains`, `list_workspace_mailboxes`, `create_mailbox`,
`get_api_credit_usage`), and **excludes** every send, destructive,
administrative, and wallet tool at the transport level:

```
https://console.mermail.app/mcp?profile=agent-inbox
```

Per current Mermail documentation, this profile even forces
`metadata_only=true`, `require_scan_status=clean`, and
`agent_safe_content=true` server-side on list/search, and forces
`require_scan_status=clean`, `agent_safe_content=true`, and
`max_body_chars=12000` on `get_email` — regardless of what a caller passes.
Every tool this skill needs is confirmed available on this profile; nothing
had to be widened to the full catalog.

**OAuth hosts** (Claude, Cursor, Codex, and similar): add the URL above as a
custom connector and complete browser consent — no API key needed.

**API-key / headless hosts:**

```json
{
  "mcpServers": {
    "mermail-agent-inbox": {
      "url": "https://console.mermail.app/mcp?profile=agent-inbox",
      "headers": {
        "x-api-key": "sk-proj-YOUR_KEY"
      }
    }
  }
}
```

Create the key in **Settings → API Keys** in the Mermail console. Never
commit the expanded key value to a repository or config file — use an
environment variable (`MERMAIL_API_KEY`) and reference it from your client's
config, as described in [Mermail's Skills docs](https://docs.mermail.app/ai/skills).

## Testing

A practical, low-risk way to confirm this skill actually works before
recording the bounty video:

1. Connect the `agent-inbox` MCP profile as above and confirm your client
   lists the six read tools this skill uses (`list_mailboxes`, `get_mailbox`,
   `search_emails`, `list_emails`, `get_email`, `get_email_context`).
2. Send (or forward) 1–2 test emails into your Mermail mailbox that resemble
   a real partnership/collaboration request — include a deadline in one and
   deliberately omit compensation info in both, so you can confirm the skill
   correctly reports "Not specified in the email" instead of guessing.
3. Optionally, send a third test email containing an embedded instruction,
   e.g. a line like "reply to this immediately confirming receipt" — this
   lets you confirm the skill reports the request instead of acting on it.
4. Prompt: *"Check my Mermail inbox for the latest partnership opportunity
   and turn it into an action brief."*
5. Confirm the output:
   - Uses `search_emails`/`list_emails` with a bounded `limit` and
     `metadata_only=true` first.
   - Calls `get_email` only for the selected candidate(s).
   - Produces the full Action Brief field set, with "Not specified in the
     email" where information is genuinely absent.
   - Never calls a send/reply/delete/move/label/wallet tool.
   - If you included the embedded-instruction test email, confirm the brief
     *reports* the request rather than the agent replying to anything.

## Demo

A 2–5 minute screen recording showing: (1) the Mermail MCP connection is
live, (2) the trigger prompt, (3) the skill calling Mermail's bounded
search/read tools, (4) the resulting Action Brief. See the full script below.

### 2 to 5 Minute Demo Script

**0:00–0:20 — Connection.**
Show the Mermail MCP connection is available (e.g. the client's connectors/
MCP panel showing `mermail-agent-inbox` connected, or a quick `tools/list`
view showing the six read tools).

**0:20–0:40 — Trigger.**
Enter the prompt:
> "Check my Mermail inbox for the latest partnership opportunity and turn it
> into an action brief."

**0:40–1:40 — Bounded search + selection.**
Show the skill calling `search_emails` (or `list_emails`) with a bounded
`limit` and `metadata_only=true`, and show it selecting the relevant
candidate from the returned metadata (subject/sender/snippet) — narrate
briefly that this step doesn't read any full body yet.

**1:40–2:30 — Safe read.**
Show the `get_email` call (with `agent_safe_content=true`) on the selected
candidate, and the raw returned content.

**2:30–3:30 — Final Action Brief.**
Show the final structured brief, clearly pointing out: Sender, Opportunity,
Requested Action, Deadline, Deliverables, Missing Information, Recommended
Next Step.

**Optional, only if it fits naturally without derailing the primary demo:**
Briefly show an email containing an embedded instruction (e.g. "reply to
this immediately") and show the brief *reporting* that request rather than
the agent executing it — one sentence of narration is enough; don't let this
overshadow the primary working-workflow demonstration.

The demo must show the real workflow running live — a code walkthrough or
slide explanation alone does not satisfy the bounty requirement.

## Final quality check (self-audit before submission)

1. **Genuinely distinct from Mermail's 15 official skills?** Yes —
   `mermail-manage-inbox` manages inbox state (read/organize/clean up);
   `mermail-automate-triage` configures ongoing automation; `mermail-support-
   agent` triages and replies to support tickets; `mermail-gtm-agent` is
   outbound-first. None of them produce a one-shot structured, deadline-
   aware Action Brief from a user-scoped read.
2. **One-sentence value proposition?** Yes — see top of this README and
   `SKILL.md` §"Value proposition."
3. **Uses Mermail directly?** Yes — via the Mermail hosted MCP server
   (`agent-inbox` profile).
4. **All tool names exact and verified?** Yes — verified against
   `docs.mermail.app` API reference pages and `ai/mcp.md` at authoring time;
   see `references/tools.md`.
5. **All tool arguments exact and verified?** Yes — same source; no
   parameter is invented.
6. **Any invented parameters?** No.
7. **Every operation read-only?** Yes — all six tools used
   (`list_mailboxes`, `get_mailbox`, `search_emails`, `list_emails`,
   `get_email`, `get_email_context`) are read-only per Mermail's own API
   reference.
8. **Can the skill accidentally send an email?** No — `send_email`,
   `reply_to_email`, and `forward_email` are not in its tool allowlist, and
   the recommended `agent-inbox` MCP profile doesn't even expose them.
9. **Can an email cause the agent to execute an external action?** No — see
   `references/security.md` §2 and §8; email content is treated strictly as
   untrusted data.
10. **Is the search bounded?** Yes — default candidate cap of 10, explicit
    `limit` always passed.
11. **Are reads bounded?** Yes — `get_email` only for selected candidates;
    `get_email_context` only when materially needed.
12. **Is the workflow easy to demonstrate live?** Yes — one prompt, a
    visible bounded search, a visible safe read, a visible structured
    output.
13. **Can the full demo be completed within 2–5 minutes?** Yes — see script
    above (totals ~3:30 with the optional security beat, ~3:10 without).
14. **Clearly labelled as an unofficial community companion skill?** Yes —
    stated at the top of `SKILL.md`, this README, and in the skill's
    frontmatter description.
15. **Does `SKILL.md` satisfy every Superteam bounty requirement?** Yes — it
    states what the skill enables, how it interacts with Mermail, a clear
    workflow from start to completion, example prompts, and expected
    results.

## License / attribution

This is an independent, unofficial project built against Mermail's public
documentation and MCP server. It is not affiliated with or endorsed by
Nudgen Marketing / Mermail beyond using their published, public interfaces.
