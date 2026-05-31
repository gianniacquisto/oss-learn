# Contributing a Course: Methodology Guide

This document describes how to create a high-quality course for oss-learn. Every course should follow this structure and philosophy.

## The Core Principle

**You are the expert.** Your job is not to auto-generate content — it's to distill a real project into its essential learning path. Use your understanding of the source project to decide what matters, what to cut, and how to tell the story.

---

## Course Structure

Every course lives under `courses/<project-slug>/` and follows this structure:

```
courses/<project-slug>/
├── README.md           # Course landing page (see template below)
├── milestones/         # Each milestone = a working increment
│   └── <N>-<name>/
│       ├── README.md   # What you'll build + why it matters
│       ├── context.md  # Background knowledge for this milestone
│       ├── steps.md    # Detailed instructions (hybrid format)
│       └── done.md     # Verification criteria — "if X works, you're good"
├── glossary.md         # Project-specific terms + definitions + external links
├── key-concepts.md     # The big ideas this project teaches
├── reading-list.md     # Curated external resources per topic area
└── notes/              # Your research material (git history, docs analysis)
```

---

## Writing Milestones

### How Many Milestones?

Aim for **4-7 milestones** per course. Fewer than 4 feels like a tutorial; more than 7 loses momentum. Each milestone should take the learner **30 minutes to 2 hours**.

### What Makes a Good Milestone?

1. **Tangible deliverable** — at the end, the learner has something runnable (a working module, a CLI command that responds, etc.)
2. **Clear "why"** — they understand why this piece exists in the larger architecture
3. **Builds on the previous** — each milestone adds to what came before, never starts from scratch

### The Hybrid Instruction Format

Use this structure for `steps.md` within each milestone:

```markdown
## Step N: <Title>

**Goal:** What you're building and why it matters in one sentence.

**Requirements:**
- Requirement 1 (what the code must do)
- Requirement 2
- Requirement 3

**Hints:**
- Conceptual guidance — "think about X before Y"
- Pattern suggestions — "consider using Z for this"
- Common pitfalls to avoid

**Reference pattern:**
```typescript
// Structural example showing API shape or key decisions.
// Don't copy-paste — write your own version from scratch.
export function myFunction(params): ReturnType { ... }
```

**Verify:** Concrete check — "Run `./myapp --help` and see X output."
```

### Calibrating Difficulty Per Milestone

| Phase | Hints | Reference Pattern | Verification |
|---|---|---|---|
| Early (setup, scaffolding) | More explicit — file structure, import paths | Full API signatures + key type definitions | `npm run build` succeeds |
| Middle (core logic) | Conceptual hints only | Structural pattern showing the flow | Run a test / CLI command |
| Late (integration, polish) | Minimal — edge cases to consider | None or pseudocode only | Integration test passes |

### The Golden Rule

> **Never write code in `steps.md` that the learner could copy-paste and be done.** If they can do it without thinking, move it to a hint or remove it entirely. The reference pattern should show *what* not *how*.

---

## Writing README.md (Course Landing Page)

Include:
- **What you'll build** — clear description of the final product
- **Difficulty** — Beginner / Intermediate / Advanced
- **Time estimate** — realistic range (e.g., "8-12 hours")
- **Prerequisites** — what they should already know (languages, tools, concepts)
- **What you'll learn** — 3-5 bullet points of key takeaways
- **Milestone overview** — brief list of milestones with one-line descriptions

---

## Writing glossary.md

For each term:
1. **Term name** in bold
2. **Definition** in plain language (not a dictionary copy)
3. **Why it matters here** — how this concept applies to the project being built
4. **External link** — MDN, TypeScript docs, RFC, etc.

Example:
```markdown
### MCP (Model Context Protocol)
A protocol that standardizes how AI agents interact with external tools and data sources. Instead of each agent having custom integrations, MCP provides a shared interface. In this project, we use it so the agent harness can talk to any LLM provider without rewriting core logic.

[Read more](https://www.modelcontextprotocol.io/)
```

---

## Writing key-concepts.md

This is the "big picture" section. Cover:
- **Architecture patterns** used in the source project (e.g., dependency injection, event loop, plugin system)
- **Trade-offs** the original authors made and why
- **Design decisions** that shaped the project's evolution
- **What makes this project interesting** — what would a learner take away from studying it?

This is where your expertise as an expert shines. Don't just describe — explain *why* things are the way they are.

---

## Writing reading-list.md

Group resources by topic area within the course:
- "TypeScript patterns" → links to relevant articles, docs, videos
- "Agent architecture" → papers, blog posts, talks
- "LLM integration" → API docs, comparison guides, best practices

Curate aggressively. 5 great links beat 20 mediocre ones. Add a one-line note about why each link is worth reading.

---

## Using notes/ for Research

This directory is your research workspace. Put here:
- Git history analysis ("here's how the agent loop evolved across commits")
- Docs you referenced while planning
- Architecture diagrams or sketches
- Decisions you made about scope and why

You pull insights from `notes/` when writing `context.md`, `key-concepts.md`, etc. The learner doesn't need to read this — it's your working material that informs the course quality.

---

## Checklist Before Publishing a Course

- [ ] Every milestone has a tangible, verifiable deliverable
- [ ] Instructions use the hybrid format (requirements + hints + structural reference)
- [ ] No step contains code that can be copy-pasted to complete it
- [ ] Verification criteria are concrete and testable
- [ ] Glossary terms explain *why they matter* in context, not just definitions
- [ ] Key concepts section explains the "why" behind architectural decisions
- [ ] Reading list is curated (not a dump of search results)
- [ ] Time estimate feels realistic for the stated difficulty level
- [ ] Prerequisites are specific and honest
