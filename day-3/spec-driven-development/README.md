# Spec Driven Development

**Day 3, Session 7 — Victor, Pasha & Ina**

*AI-driven spec and test strategy — greenfield, brownfield, and source of truth (Wed, 29 Jul 26)*

## Greenfield vs. Brownfield AI Strategy

- Greenfield: use AI for research, web search integrations, coding agents to generate specs
- Brownfield: use AI to build the initial spec from existing code/UI
- Either way, AI drives spec generation; the approach differs, not the tooling philosophy

## Source of Truth: A Human Problem, Not a Technical One

- No single right answer: code, spec, docs (Confluence, Jira, ADRs, PRDs) all viable
- Choice depends on what the organization already uses and who owns it
- Key principle: pick an approach and follow it consistently
- AI can automate the process of keeping sources of truth in sync across tooling

## Spec-to-Test Pipeline (Intent Integrity Chain)

- Describe intent as acceptance criteria, then generate deterministic tests from them
- BDD/Gherkin syntax used to match spec scenarios to test code
- Recommendation: avoid Cucumber JVM overhead; use Gherkin notation with traditional JVM + Spock
- Intent Integrity Kit already published on GitHub: worth checking out
- Inspired by Baruch Sadagorsky's talk from JSpring 2026 ("Monkey" something, keynote length, short)
- Plans (long, code-heavy) rarely read; specs and executive summaries more practical at scale
- AI can generate exec summaries of large plans when full reading is impractical

## Victor's Live Demo: Spec-Driven Voting App

- Workflow: ChatGPT/Claude/Groq for research, spoke intent aloud, AI extended it into a structured spec (spec.md)
- Dropped spec into Claude Code, used Beans to capture milestones
  - Milestone 1: start spike
  - Milestone 2: review and publish
  - Milestone 3: voting live results and publish
- Beans: local issue tracker living inside the project folder, markdown-based with front matter
  - Replaces heavyweight trackers for solo/small projects
  - Issue tracker as source of truth: update beans items as work progresses
- App extracted topics from a photo of the whiteboard, generated a QR-code voting interface
- Deploying to Railway (localhost too tricky for live demo); URL to be shared via WhatsApp
- Minor bug: "publish results" not triggering until voting is closed first

## Key Takeaways

- Communicating intent clearly is the most important skill now: for people and for machines
- Structure the big idea first; everything else (plan, tasks, tests) flows from the spec
- Hierarchy: spec (what features) → plan (user stories/steps) → tasks (developer assignments)
- AI productivity gains most visible on greenfield: one engineer covered a complex spike solo with Claude, full test suite passing, ~80% efficiency gain vs. ~25% on legacy projects
