# Aira — Architecture Overview

**Production:** [researchwithaira.com](https://researchwithaira.com)

This document describes how Aira is designed at a high level. Implementation details and source code are in a private repository.

---

## Design principles

1. **Teams over individuals** — Plans persist, version, and support multi-user collaboration with roles and invites.
2. **Structure over summary** — Output is a typed collaboration plan (goals, decisions, gaps, glossary), not a prose paragraph.
3. **Interdisciplinarity by design** — Agents explicitly model disciplines, perspectives, missing voices, and cross-field terminology.
4. **Human-in-the-loop** — Follow-up meeting updates produce a diff for review; users approve before anything is merged.
5. **Server-side trust** — Authorization is enforced on the backend, not only in the UI.

---

## System layers

```
┌─────────────────────────────────────────────────────────────────┐
│                         Browser (React SPA)                     │
│   Input modes │ Plan viewer │ Merge review │ Collaboration UI   │
└───────────────────────────────┬─────────────────────────────────┘
                                │ HTTPS + authenticated requests
                                ▼
┌─────────────────────────────────────────────────────────────────┐
│                      Application server (FastAPI)               │
│  Transcription │ Agent orchestration │ REST API │ Auth checks   │
└───────────────────────────────┬─────────────────────────────────┘
                                │
              ┌─────────────────┴─────────────────┐
              ▼                                   ▼
       OpenAI API                           Supabase
    (Whisper + GPT-4o)              (Auth + PostgreSQL + RLS)
```

| Layer | Responsibility |
|---|---|
| **Frontend** | Transcript input, plan visualization, merge review, notes, invites |
| **Backend** | Transcription, agent pipeline, persistence, role enforcement |
| **Supabase** | User auth (Google OAuth), relational storage, row-level security |
| **OpenAI** | Speech-to-text and all LLM agent calls |

---

## End-to-end data flow

```
User submits meeting content
        │
        ▼
┌───────────────────┐
│  Transcription    │  Audio/video → text (Whisper + ffmpeg)
└─────────┬─────────┘
          ▼
┌───────────────────┐
│  Agent pipeline   │  Transcript → structured CollaborationPlan
└─────────┬─────────┘
          ▼
┌───────────────────┐
│  Persistence      │  Plan saved to database; user sees tabbed UI
└─────────┬─────────┘
          ▼
┌───────────────────┐
│  Collaboration    │  Status updates, notes, invites, version history
└───────────────────┘
```

---

## Multi-agent pipeline

Plan generation is staged: one agent runs first to extract shared context, then three agents run in parallel.

### Initial plan generation

```
                    Transcript
                        │
                        ▼
              ┌─────────────────┐
              │  Agent 1        │
              │  Metadata       │── disciplines, goals, resources
              └────────┬────────┘
                       │
         ┌─────────────┼─────────────┐
         ▼             ▼             ▼
   ┌──────────┐  ┌──────────┐  ┌──────────┐
   │ Agent 2  │  │ Agent 3  │  │ Agent 4  │
   │Perspect- │  │Reasoning │  │ Glossary │
   │  ives    │  │          │  │          │
   └────┬─────┘  └────┬─────┘  └────┬─────┘
        │             │             │
        └─────────────┼─────────────┘
                      ▼
            CollaborationPlan
```

| Agent | Role | Key outputs |
|---|---|---|
| **1 — Metadata** | Frame the meeting | Disciplines present, research goals, people/tools/datasets |
| **2 — Perspectives** | Cross-disciplinary synthesis | Viewpoints per domain, flagged gaps (missing voices) |
| **3 — Reasoning** | Actionable outcomes | Decisions (with dissent & tradeoffs), next steps, assumptions |
| **4 — Glossary** | Shared vocabulary | Terms, definitions, domain tags, related terms |

**Agent 2 enrichment:** Before synthesizing perspectives, a domain enrichment step identifies genuinely missing disciplinary voices based on transcript evidence — optionally powered by a fine-tuned model trained on interdisciplinary research scenarios.

All agents output validated structured JSON against a shared schema contract.

### Follow-up meeting flow

When a team uploads a transcript from a subsequent meeting:

```
Existing plan (v1)              New transcript
       │                              │
       │                         Agents 1–4
       │                              │
       └──────────────┬───────────────┘
                      ▼
              Update agent (diff)
         Returns only net-new items
                      │
                      ▼
           User reviews in UI
         (approve / reject per section)
                      │
                      ▼
              Merge → plan v2
```

Nothing is auto-overwritten. The update agent compares the new synthesis against the current plan; the UI presents a diff; approved items are appended into a new version.

---

## Collaboration plan schema

The central data object is a **CollaborationPlan** — a typed JSON document assembled by the agents and rendered in the UI:

```
CollaborationPlan
├── Metadata        plan ID, version, disciplines, timestamp
├── Goals           statement + priority (high / medium / low)
├── Perspectives    domain + viewpoint
├── Flagged gaps    missing voice + impact
├── Assumptions     explicit or inferred
├── Resources       people, tools, datasets
├── Decisions       status, rationale, dissent, tradeoffs
├── Next steps      action, domain, due date, status (open / done / blocked)
└── Glossary        term, definition, domain, related terms
```

---

## Data model

Plans are organized hierarchically to support versioning and team access:

```
Organization
    └── Project
            ├── Members        (roles: admin, editor, viewer)
            ├── Invites        (token-based onboarding)
            └── Plan
                    └── Version  (v1, v2, …)
                            ├── Goals
                            ├── Perspectives & gaps
                            ├── Decisions & assumptions
                            ├── Next steps & notes
                            └── Glossary terms
```

Each plan version is immutable once created. Follow-up merges produce a new version rather than editing history.

Row Level Security ensures users only access projects they belong to.

---

## Frontend

Single-page React application with two primary modes:

1. **Input** — paste transcript, upload file, or record live in the browser
2. **Plan viewer** — tabbed sections for each plan component, plus collaboration tools

**Collaboration features in the plan viewer:**
- Role-aware editing (viewers are read-only)
- Next-step status tracking (`open` / `done` / `blocked`)
- Per-step and summary notes (persisted per user)
- Invite teammates via link
- Merge diff review for follow-up meetings
- Version history navigation
- Export to CSV or JSON

Authentication uses Google OAuth via Supabase. An optional access code gate supports invite-only rollout.

---

## Authentication and roles

```
User → Google OAuth → Supabase session (JWT)
                              │
                              ▼
                    Every API request authenticated
                              │
                              ▼
                    Server checks project membership + role
                              │
              ┌───────────────┼───────────────┐
              ▼               ▼               ▼
           admin           editor          viewer
     invites, members   edit, merge      read-only
```

**Invite flow:** An admin generates an invite link. The invitee signs in with Google and accepts the token to join the project with the assigned role.

---

## Deployment

Production runs as a single server process that handles both API requests and the pre-built frontend — one origin, no CORS complexity.

```
┌────────────────────────────────────┐
│  Cloud host                        │
│  ┌──────────────────────────────┐  │
│  │  FastAPI                     │  │
│  │  • REST API                  │  │
│  │  • Serves React SPA          │  │
│  └──────────────────────────────┘  │
│  ffmpeg installed for video input  │
└────────────────────────────────────┘
         │                │
         ▼                ▼
    OpenAI API      Supabase Cloud
```

DNS: `researchwithaira.com` → production host.

---

## Evaluation methodology

Aira is benchmarked against a **baseline**: a single LLM prompt that produces an unstructured prose summary of the same transcript.

An **LLM-as-judge** scores both outputs on six criteria:

| Criterion | What it measures |
|---|---|
| Completeness | Coverage of meeting content |
| Structure | Organization and navigability |
| Actionability | Clear next steps and decisions |
| Interdisciplinary coverage | Representation of multiple domains |
| Gap identification | Missing perspectives surfaced |
| Glossary quality | Useful cross-disciplinary definitions |

Each criterion is scored 0–5 (max 30). This framework validates that the multi-agent structured approach adds value beyond generic summarization.

---

## Key design decisions

| Decision | Rationale |
|---|---|
| Parallel agents after metadata | Agents 2–4 are independent once disciplines are known — reduces latency |
| Diff-based updates | Preserves user trust; teams control what changes between meetings |
| Versioned plans | Audit trail of how a research direction evolved across meetings |
| Dual storage (JSON + relational) | Fast full-plan reads plus granular updates (status, notes) |
| Monolith deploy | Simpler ops for a focused product; API and UI share one origin |

---

## Related

- [README.md](./README.md) — product overview, features, team
- [researchwithaira.com](https://researchwithaira.com) — live application
