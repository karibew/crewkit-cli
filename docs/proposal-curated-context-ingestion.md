# Proposal: Curated Context Ingestion ("the company brain")

> **Status:** 🟡 Proposal — for team reaction. Nothing here is decided.
> **Date:** 2026-06-04 · **Author:** Arthur Genest
> **Origin:** Brainstorm during the 2026-06-04 dev sync, sparked by the team-agent UI demo (the screen where you set an agent's email + Slack ID). Participants: Arthur Genest, Félix-Antoine Paradis, Rémi Gagnon.
> **Related:** [PLAN.md](../PLAN.md) Phases B–D · [meeting-tasks-2026-06-04.md](./meeting-tasks-2026-06-04.md) Epics B/D/F · `feat/session-vectorizer` (the F-side machinery, in review)

---

## The one-paragraph pitch

Give each project a crewkit agent with its own **email address and Slack presence**. When someone has information worth remembering — a decision in an email thread, meeting notes, a Slack discussion of a business rule — they **CC the agent** or **flag the Slack thread to it**. The agent vectorizes only what it's handed and adds it to the project's knowledge base, where it becomes searchable context for every other agent and teammate. The org decides what the brain learns, one deliberate forward at a time.

## Why this shape (the strategic bet)

The obvious alternative is the **"connect everything" model** (Glean-style): OAuth into Slack + Gmail, index the whole workspace. It's powerful but it's heavy:

- Needs org-wide permissions → an IT/security review before *anyone* gets value.
- Feels like surveillance to employees; ingests salary talk, HR threads, venting.
- Pollutes the knowledge base with noise (the #1 failure mode of retrieval systems).

**Our bet:** opt-in curation is a feature, not a compromise.

| | Connect-everything | Curated CC/invite (this proposal) |
|---|---|---|
| Permissions | Org-wide OAuth scopes | None — just add a contact / channel member |
| Time-to-value | After security review | Day one, bottoms-up |
| KB signal quality | Noisy, needs filtering | Clean — humans are the filter |
| Privacy posture | "It reads everything" | "It only knows what we told it" |

This is a genuine wedge for small teams who'd never pass (or wait for) an enterprise integration review.

**The honest risk of this bet** (see Open Question 5): opt-in's strength is clean data; its weakness is *recall* — the brain only knows what someone remembered to forward. If we don't reliably feed it, neither will customers.

---

## How it would work

```
Someone CC's an email  ─┐
Someone flags a Slack  ─┼─►  Inbound webhook  ──►  Verify sender  ──►  Sanitize  ──►
thread to the agent    ─┘   (SendGrid / Slack)     (is an org member?)

  ──►  Create Artifact (source: email/slack)  ──►  Vectorize  ──►  Project Knowledge Base
                                                   (existing pipeline)        │
                                                                              ▼
                                              Searchable context for all agents + "Find prior work"
```

The vectorize → KB half already exists (`feat/session-vectorizer`). This proposal is the **capture front door** that feeds it.

> **Design note:** capture is a *dumb pipeline*, not an LLM "agent." The email/Slack identity is the friendly interface; the backend is webhook → sanitize → Artifact → existing vectorizer. We spend LLM tokens *downstream* (routing, dedup, contradiction detection), never at capture.

---

## Decisions we need the team to react to

### ❓ Decision 1 — Per-project address, or one per org?
- **Per project** (`projname@acme.crewkit.io`): the address *is* the routing key — solves "which project does this belong to?" for free. More addresses to manage.
- **Per org** (one brain): simpler to share, but needs a routing step (auto-guess + a dashboard "unassigned" tray) to file content to the right project.
- **Leaning:** per-project for v1. Routing is the hardest unsolved problem from the meeting; per-project addressing sidesteps it.

### ❓ Decision 2 — The auto-generated identifier in the address
The meeting flagged that orgs could collide on agent names, so addresses should include an auto-generated ID (e.g. `name-<id>@crewkit.io`).
- **Reframe:** collision is the *weak* reason (solved by uniqueness validation within an org namespace). The *strong* reason is **security** — an unguessable address acts as a bearer token so outsiders can't email content into your brain (see Decision 4).
- **Tension:** a random ID makes the address un-typeable. So **if** we use unguessable addresses, we **must** ship the VCF contact card (Epic D3) so humans pick the contact instead of typing it. The two ship together or not at all.

### ❓ Decision 3 — Slack is NOT symmetric with email
- Email = **CC** (one explicit message, whole thing). Clean.
- Slack "**add to channel + track everything**" reopens the surveillance problem we're trying to avoid.
- **Proposal:** Slack capture is **trigger-based** — react with an emoji (`:brain:`) or `@crewkit capture this thread` — not channel-membership-based. Preserves per-item curation; captures the whole thread, not one line.
- **Decision:** do we accept that email and Slack have *different* interaction models?

### ❓ Decision 4 — Sender verification & poisoning (security gate)
If the brain feeds other agents, whoever writes to the brain can manipulate the agents.
- **Spoofing:** email `From` is forgeable → must verify sender is a known org member + enforce SPF/DKIM/DMARC.
- **Indirect prompt injection:** ingested text is later retrieved into agent context. A "meeting note" saying *"ignore previous instructions…"* becomes an attack. Retrieved context must be treated as **untrusted data, never instructions**.
- **Action:** `security-expert` review before this ships. Not a blocker on the idea — a constraint on the build.

### ❓ Decision 5 — Governance: who can teach the brain, and is its word law?
- Any employee CC-ing content means any employee shapes what every agent "knows." A junior's wrong assumption can become cited "fact."
- **Proposal:** ingested context is a **lower trust tier** ("things people said") than playbooks/conventions ("rules we enforce"). It informs; it never silently overrides governance.
- Requires: provenance per chunk (who/when/source — we have `uploaded_by_id` + metadata), and easy **retraction** (extend the `retract` we built for sessions) for the "that had confidential data" case.

---

## What we'd build (maps to existing roadmap)

| Stage | Epic | Status |
|---|---|---|
| Capture front door (SendGrid inbound, Slack trigger, VCF card, sender verification) | **D** | Phase B, in flight |
| Routing (per-project address, or auto-guess + unassigned tray) | **B** | Not started |
| Vectorize → KB → searchable | **F** | ✅ Landed (`feat/session-vectorizer`) |
| Contradiction detection on ingested decisions | **F3** | Next — the headline follow-on |

This is not a new project — it's the **product vision that ties B + D + F together.**

---

## Open questions for the room

1. Per-project vs per-org addressing? (Decision 1 — the one to nail first.)
2. Are we OK with email/Slack being asymmetric? (Decision 3)
3. Trust tier for ingested context vs governance — informs-only, or can it override? (Decision 5)
4. Who owns the SendGrid + sender-verification + injection-hardening work? (Decision 4)
5. Recall problem: how do we make sure *we* actually feed it (dogfooding), and do we want a "you haven't filed anything to Project X lately" nudge?

---

*This is a proposal to react to, not a plan to execute. Tear it apart.*
