# Aira

**AI research assistant for interdisciplinary scientific teams**

🌐 **Live app:** [researchwithaira.com](https://researchwithaira.com)

> Source code is private. This repository is the public overview — architecture, features, and team. For access inquiries, open an issue or contact the team.

---

## The problem

Scientific discovery increasingly depends on interdisciplinary teams whose members bring distinct expertise, vocabularies, assumptions, and standards of evidence. Today's AI research assistants optimize individual workflows: literature review, writing, coding but offer little support for the **collaborative reasoning** required to integrate knowledge across disciplines.

## What Aira does

Aira turns research meeting recordings into **structured collaboration plans**. Upload audio or video, record live in the browser, or paste a transcript and receive an actionable plan that surfaces what matters for cross-disciplinary work:

- **Goals** and prioritized objectives
- **Decisions** with rationale, dissent, and tradeoffs
- **Next steps** with owners, domains, and status tracking
- **Perspectives** synthesized from each discipline present
- **Flagged gaps** where missing voices could change the research direction
- **Assumptions** — explicit and inferred
- **Cross-disciplinary glossary** — shared vocabulary across fields

Aira is designed for teams, not solo researchers: plans persist, evolve across follow-up meetings, and support role-based collaboration.

---

## Key features

| Feature | Description |
|---|---|
| **Multi-modal input** | Paste a transcript, upload audio/video, or record in-browser |
| **Multi-agent synthesis** | Four specialized agents extract metadata, perspectives, reasoning, and glossary terms in parallel |
| **Follow-up meetings** | Upload a new transcript; Aira diffs against the existing plan for human review before merging |
| **Version history** | Each approved merge creates a new plan version |
| **Team collaboration** | Roles (`admin`, `editor`, `viewer`), invite links, persisted notes and action status |
| **Human-in-the-loop** | Users approve or reject suggested updates — nothing is auto-overwritten |

For system design details, see [Architecture.md](./Architecture.md).

---

## How it works (at a glance)

```
Meeting input  →  Transcription  →  Multi-agent pipeline  →  Collaboration plan
   (text /              (Whisper)         (4 LLM agents)         (structured JSON
    audio /                                              + collaborative UI)
    video /
    live record)
```

**Agent pipeline:**
1. **Metadata** — disciplines, goals, resources
2. **Perspectives** — viewpoints and flagged gaps *(runs in parallel)*
3. **Reasoning** — decisions, next steps, assumptions *(runs in parallel)*
4. **Glossary** — cross-disciplinary term definitions *(runs in parallel)*

A fifth **update agent** handles follow-up meetings by diffing a new plan against the existing one.

---

## Tech stack

| Layer | Technologies |
|---|---|
| Frontend | React, Vite, Tailwind CSS, shadcn/ui |
| Backend | Python, FastAPI |
| Database & auth | Supabase (PostgreSQL, Google OAuth, Row Level Security) |
| AI | OpenAI Whisper (transcription), GPT-4o (agents), fine-tuned domain model (perspective enrichment) |
| Deployment | Single-process monolith (API + SPA), ffmpeg for video audio extraction |

---

## Evaluation

Aira is evaluated against a single-prompt baseline (unstructured prose summary) using an LLM-as-judge on six criteria — completeness, structure, actionability, interdisciplinary coverage, gap identification, and glossary quality; scored 0–5 each (max 30).

The multi-agent structured output consistently outperforms the baseline on actionability and cross-disciplinary synthesis.

---

## Research

Aira is developed as part of interdisciplinary AI research at Duke University. The system is described in:

> **Aïra: Rethinking AI Research Assistants for Interdisciplinary Science**  
> Mirji, Degbotse, Nanjappa, Zhou, et al. — Duke University

*Paper link to be added when published.*

---

## Team

Built at the **Duke Trust Lab** (Pratt School of Engineering, Duke University).

| | |
|---|---|
| **Dr. Brinnae Bent** | Faculty advisor |
| **Sharmil Nanjappa** | Engineering |
| **Diya Mirji** | Engineering |
| **Tiffany Degbotse** | Engineering |
| **Jiayi Zhou** | Engineering & Environment |


---

## Links

- 🌐 [researchwithaira.com](https://researchwithaira.com) — live application
- 📐 [Architecture.md](./Architecture.md) — system design overview
- 🔒 Source code — private repository (not open source)

---


