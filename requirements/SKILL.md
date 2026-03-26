---
name: requirements
description: "Generate concise, well-structured feature requirement documents optimized for Claude Code Plan mode. Use this skill whenever the user wants to plan, spec out, define, or document a new feature, enhancement, or change to any software project. Trigger when the user says things like 'I want to add...', 'let's build...', 'I'm thinking of a feature...', 'write up requirements for...', or 'help me spec out...'. The output is a focused requirements doc purpose-built for handing to Claude Code in Plan mode."
---

# Requirements Skill

This skill helps you write a focused requirements document for a new feature or change.
The goal is a document concise enough for Claude Code to plan implementation without drowning
in context — just enough signal to plan confidently.

## Architecture Context

If a project-specific requirements skill has loaded an Architecture Reference above this section,
use that. Otherwise, ask the user to briefly describe the relevant parts of their codebase
(services, key files, data models) before proceeding to the interview.

---

## Your Workflow

### Step 1: Interview the user

Ask only what you don't already know from context. Typically you need:

1. **What's the goal?** What problem does this feature solve, or what capability does it add?
2. **Which component(s)?** Which services, modules, or files are likely affected?
3. **Acceptance criteria** — what does "done" look like? Ask for 2–5 specific, testable outcomes.
4. **Constraints** — any patterns to follow, things to avoid, performance concerns?
5. **Non-goals** — what's explicitly out of scope for this feature?

If the user has described the feature in detail already, extract answers from context before asking.
Keep the interview short — 1–2 rounds of questions at most. Make reasonable assumptions for
anything minor and note them in the document.

### Step 2: Generate the requirements document

Write a markdown document using the structure below. Aim for 200–400 words — enough to give
Claude Code clear direction without padding. Every line should earn its place.

---

## Output Format

```markdown
# Feature: [Concise Name]

## Goal
[1–2 sentences. What problem does this solve? Why does it matter?]

## Affected Components
- [Component name] — [one-line summary of what changes]
- [File path if known] — [what specifically changes there]

## Acceptance Criteria
- [ ] [Specific, testable outcome — user-facing or measurable]
- [ ] [Another testable criterion]
- [ ] [Etc. — aim for 3–5]

## Constraints
- [Technical pattern to follow]
- [Performance or correctness constraint]
- [Anything that must not change]

## Non-Goals
- [Explicitly excluded scope item]
- [Another exclusion]

## Notes / Assumptions
[Any assumptions made during spec-writing, or background context Claude Code will need
that isn't obvious from the codebase — e.g., quirks of an external API, known limitations
of the current design, or dependency considerations.]
```

---

## What Makes a Good Requirements Doc for Plan Mode

Claude Code's Plan mode works best when requirements are:

- **Problem-oriented, not solution-oriented** — describe *what* to achieve, not *how* to implement it. Let Claude Code figure out the implementation.
- **Specific about outcomes** — vague criteria like "improve performance" are hard to plan against. "Response time under 200ms for `/api/stories`" gives Claude Code something to target.
- **Explicit about scope** — non-goals prevent Claude Code from over-engineering or making assumptions about what's in scope.
- **Grounded in the codebase** — mention affected files/services by name so Claude Code starts exploring in the right place.
- **Short** — Claude Code works better with focused context. Resist the urge to explain how things currently work in great detail; Claude Code will read the code itself.

---

## Example

**User says**: "I want to add rate limiting to the API."

**Good requirements doc**:

```markdown
# Feature: API Rate Limiting

## Goal
The public API has no request limits, making it vulnerable to abuse and accidental overload.
Add per-client rate limiting so the service remains stable under heavy or malicious traffic.

## Affected Components
- `ApiService` / `Program.cs` — add rate limiting middleware
- Configuration — new `RateLimit:RequestsPerMinute` setting

## Acceptance Criteria
- [ ] Requests exceeding the limit receive HTTP 429 with a `Retry-After` header
- [ ] Default limit is 60 requests/minute per IP, configurable via `RateLimit:RequestsPerMinute`
- [ ] Requests within the limit are unaffected in latency
- [ ] Limit resets on a sliding window, not a fixed clock boundary

## Constraints
- Use the built-in ASP.NET Core rate limiting middleware (not a third-party library)
- Follow the existing Options pattern for configuration
- Do not affect health check endpoints

## Non-Goals
- Per-user or authenticated rate limiting
- Distributed rate limiting across multiple instances
- Admin UI or dashboard for monitoring limits

## Notes / Assumptions
- Single-instance deployment assumed; a distributed counter is not needed
- IP-based limiting is sufficient for now
```
