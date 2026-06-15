# Spike C3 — Meeting-bot transcript ingestion

> **Status:** spike / recommendation. Owner: Arthur. Date: 2026-06-09.
> Source: [meeting-tasks-2026-06-04.md](./meeting-tasks-2026-06-04.md) Epic C (C1/C2/C3).
> **Decision (taken):** start with Option A (post-meeting webhook). Build it so an
> auto-join agent (Option B) can be added later without rework.

## Question

When a meeting ends, how does its transcript get into crewkit? Two paths:

| | **Option A — Post-meeting webhook** ✅ chosen v1 | **Option B — Auto-join agent** |
|---|---|---|
| How | A transcription provider (Fireflies / Otter / Gemini / Zoom) POSTs the finished transcript to a crewkit webhook | A crewkit bot joins the live call, records, transcribes itself |
| Build | thin webhook → Artifact → existing Epic B funnel | calendar integration + bot infra + live audio + transcription + storage |
| Reliability | provider owns the hard part | we own join/record/transcribe — many failure modes |
| Latency | minutes after the call | live |
| "Creepiness" | low (provider already in the call) | high (a crewkit bot visibly in every meeting) |
| Cost | provider's | our compute + transcription |

## Why A first

A delivers ~90% of the value (searchable meeting context, routed to the right project) for
~10% of the work, by reusing what already shipped:

```
transcript webhook  →  Artifact (source: "meeting")  →  ProjectRoutingService
                                                          →  auto-file (high confidence)
                                                          →  Inbox tray (low confidence)  →  vectorized, agent-retrievable
```

This is exactly **C2** ("on finished transcript: create an Artifact and route it through the
Epic B funnel") — the router, Inbox triage, and vectorization are done. C3's job is just the
intake edge.

## Option A — v1 spec

1. **Endpoint:** `POST /webhooks/transcripts/:provider` (signature-verified per provider,
   mirroring the existing SendGrid/Stripe webhook pattern).
2. **Adapter layer:** a small `TranscriptProvider` interface that normalizes each provider's
   payload → `{ title, participants[], transcript_text, started_at, meeting_url }`. One
   adapter per provider (start with **Fireflies**). This is the seam that keeps providers
   swappable.
3. **Ingest:** create an `Artifact` (`source: "meeting"`, type `meeting_transcript`) and hand
   it to the existing routing job — high-confidence auto-files, low-confidence lands in the
   Inbox. Participants feed the routing signal (who → which project).
4. **Dedup:** key on the provider's meeting ID so re-sends don't double-file.

## Keeping Option B open (the user's intent)

The design above makes B additive, not a rewrite:
- B becomes **just another `TranscriptProvider`** — a crewkit auto-join agent that produces
  the same normalized `{title, participants, transcript_text, …}` and calls the same ingest
  path. Everything downstream (routing, Inbox, vectorization) is unchanged.
- So: ship A, optionally adopt Fireflies as the provider, and only build the auto-join bot
  later if "minutes-late" or "depends on a 3rd party" becomes a real pain.

## Recommendation

**Build Option A, Fireflies adapter first, behind the `TranscriptProvider` seam.** Defer
Option B indefinitely — revisit only if live capture or provider-independence becomes a
felt need. Do **not** build bot/calendar/audio infra for v1.

## Open questions (for build time)
1. Which provider first — Fireflies (the user named it) — and does it push a webhook or must we poll?
2. Participant → project mapping: lean on existing artifact-similarity routing, or also learn from attendee emails?
3. Add `"meeting"` to `Artifact::SOURCES` (today: upload/google_drive/url/content/email/slack/session).
