# Architecture Playbook: Goals → Design → Documentation

A practical, no-fluff process for taking a project from "what are we building" to a system that stays documented as it grows.

---

## 1. Defining Goals & Requirements

Skip heavyweight requirements docs. Write a **one-page brief**, in an hour, with four sections:

- **Problem statement** — what pain does this solve, for whom?
- **Functional scope** — the 5–10 things the system must do, and explicitly what it *won't* do yet (MoSCoW — Must/Should/Could/Won't — keeps this honest)
- **Quality attributes with numbers** — not "fast," but "p95 response under 300ms"; not "scalable," but "10k concurrent users by year two"
- **Constraints** — budget, team size, timeline, compliance

> For the NGO platform, this brief should front-load "zero cost to members" and "must run on free/low-cost infra" as hard constraints — those two facts alone rule out a lot of architecture choices before a single diagram gets drawn. For ses, the equivalent constraint is probably "one codebase across Android/iOS/web/desktop, no dedicated backend team."

**Non-functional requirements** — fill this out before designing anything:

| Category        | Key question                       | Example target                         |
| --------------- | ---------------------------------- | -------------------------------------- |
| Performance     | Expected load, latency targets?    | 1,000 req/s, p95 < 300ms               |
| Scalability     | Growth expectations?               | 10x users without a rewrite            |
| Reliability     | Downtime / data-loss tolerance?    | 99.9% uptime                           |
| Security        | Auth, data protection, compliance? | OAuth2/JWT, GDPR                       |
| Maintainability | Team size, tech-debt tolerance?    | New contributor productive in < 1 week |
| Cost            | Hosting budget?                    | < $500/month at launch                 |
| Observability   | How will you debug it?             | Logs, metrics, traces                  |

**Questions worth asking** (yourself, or stakeholders in a 1-hour workshop):
- What's the worst-case load pattern — spiky or steady? What if this goes viral overnight?
- What happens if this goes down for 5 minutes? For a day? (The answer tells you whether you need real failover, or just a restart script.)
- Is slightly-delayed data okay, or does this need real-time accuracy?
- What's truly non-negotiable versus nice-to-have right now?

**Start an ADR log immediately** — before the first big decision, not after. Minimal format:

```md
# ADR 001: Use PostgreSQL instead of MongoDB
Date: 2026-07-05
Status: Accepted
Context: We need full-text search but don't want ElasticSearch overhead yet.
Decision: Use Postgres tsvector for initial search.
Consequences: No extra infra now; may become a bottleneck past ~1M rows.
```

---

## 2. Architecting the System

Don't do Big Design Up Front. Do enough to start, then evolve — apply these principles ruthlessly:

- **Separation of concerns** — presentation, business logic, and data access stay in separate layers
- **Loose coupling, high cohesion** — swap one component without ripping out its neighbors
- **Stateless services** — no session state in memory; push it to a DB/cache so scaling is just "add another instance"
- **YAGNI / evolutionary design** — build what's needed now, but don't wall off future extension
- **Fail fast + observability** — retries, circuit breakers, real logging from day one, not bolted on later

**Pick a pattern based on actual need, not ambition:**

| Pattern | Use it when |
|---|---|
| Layered monolith | Default for most small-to-mid apps — fast to build, easy to reason about, easy to deploy |
| Modular monolith | Same as above, with strict internal module boundaries, so it *can* split later without a rewrite |
| Microservices | Only with real independent-scaling needs or multiple teams stepping on each other — this solves organizational problems more than technical ones |
| Event-driven | Async workflows (notifications, background jobs), or multiple consumers reacting to the same event |
| Hexagonal / ports-and-adapters | You want core logic isolated from frameworks/tools so any one of them can be swapped later without touching business logic |

> Hexagonal is worth calling out for the NGO platform: if Phase 1 runs on Discord + Notion + Airtable + Google Forms, wrapping each behind a clean adapter interface means swapping any one out later (e.g., Airtable → a real database) never touches your core member/cohort logic.

**Design process — the C4 model** (think Google Maps: zoom in only as far as you need):

1. **Requirements brief** (section 1 above)
2. **Level 1 — System Context**: one diagram — your system, its users, the external systems it talks to
3. **Level 2 — Containers**: the deployable pieces — web app, mobile app, API, database, background workers, and how they communicate
4. **Level 3 — Components**: only for containers with real internal complexity (e.g., break the API into Auth / Billing / Notifications). Skip this for simple containers.
5. **Skip Level 4 (code)** — it goes stale the moment you write it; the code is the documentation at that granularity
6. **Write an ADR** for anything expensive to reverse: framework, database, monolith-vs-services, auth strategy
7. **Risk-driven spikes** — prototype the riskiest unknown first (e.g., the Waydroid/Capacitor Android build pipeline)

> For **ses**: Level 1 is just the user, the app, and any external quote/topic data source. Level 2 is the Vite/React frontend, Capacitor wrapping it for Android/iOS, and wherever the data lives — likely simple enough that Level 3 isn't needed yet.
>
> For the **NGO platform**: Level 1 is members (Explorers/Builders/Pioneers), volunteers, and admins. Level 2 in Phase 1 is literally Discord + Notion + Airtable + Google Forms as the "containers" — worth diagramming even pre-custom-build, so the seams are visible before anything gets replaced.

---

## 3. Documentation & Maintenance

Treat diagrams as maps, not blueprints: **code is truth**, and diagrams only earn their keep if they're cheap enough to redraw whenever things drift.

**Given a terminal-first, Fish/Arch workflow, lean text-based and git-friendly:**

| Tool | Best for | Why |
|---|---|---|
| **Mermaid** | Default choice | Plain text, lives in the repo, renders natively on GitHub/GitLab and Obsidian, scriptable via `mermaid-cli` from the terminal |
| **PlantUML** | Complex sequence diagrams | Text-as-diagram, runs as a jar — easy to wire into a Makefile/justfile |
| **Excalidraw** | Brainstorming only | Fast, free, deliberately "unfinished"-looking — not for source-of-truth docs |
| **diagrams.net** | Occasional polished diagram | Has a CLI/desktop version, useful if a diagram ever needs to go to a non-technical NGO volunteer |
| **Notion** | Non-technical collaborators only | Worth it only if NGO volunteers aren't comfortable with Git/Markdown — heavier than needed for ses |

**Living documentation — lowest-friction structure, straight in the repo:**

```
docs/
├── architecture/
│   ├── context.md
│   ├── containers.md
│   └── components/
├── adr/
│   ├── 0001-use-postgres.md
│   └── 0002-monolith-first.md
└── diagrams/        (Mermaid/PlantUML source — not just exported images)
```

**Obsidian** is worth layering on top as a personal knowledge base: local-first Markdown under the hood, syncs via Git or Syncthing with no paid plan, and its bidirectional linking is genuinely useful for connecting an ADR → the requirement that drove it → the diagram it affects.

**Keeping it alive instead of stale:**
- **One ADR per significant decision**, numbered sequentially, never deleted — mark superseded ones "superseded," don't erase history
- **Tie ADRs to PRs** — a PR that changes something architecturally significant references or adds an ADR in the same commit
- **Make diagrams disposable** — a 5-minute Mermaid edit gets kept current; a 4-hour GUI diagram doesn't
- **Quarterly (or per-milestone) review** — 30 minutes, skim the C4 diagrams and ADR log, ask "is this still true? what's hurting us?"
- **Physical + digital hybrid** — sketch loosely on paper or a whiteboard first, but any decision worth remembering becomes an ADR within 24 hours, or it doesn't count

---

### Overall
Start with a monolith with clean internal boundaries. Ship every 1–2 weeks and let the architecture earn its complexity through real pain points, not speculation. Revisit the Context/Container diagrams every 3–6 months — for two solo/small-team projects like these, that cadence is usually enough.

## [[Sohbat-e-Sadiqeen (SES)]]
