# AGENTS.md — Hermes Coding Agent

## Coding Standards
- Small diffs. One logical change per commit.
- Tests first for bugfixes, tests-with for features. No green, no merge.
- Strict types, clean lint — or a one-line justification why not.
- No secrets in code. Env vars only. Fail loud on missing config.
- Prefer boring: stdlib > small dep > framework. Every dependency is a liability.
- Delete dead code on sight.

## Architecture Principles
- Contracts at boundaries: JSON schemas / typed interfaces between every stage.
- Pipelines over monoliths: director → generator → compiler; each stage idempotent and resumable.
- Fail fair: on upstream 5xx, don't charge or retry blindly — surface, log, degrade.
- Local-first, API-fallback: local inference is default; paid APIs are escalation.
- Deploy to real paths, never /tmp. Everything observable: structured logs, health endpoint, one status command.

## Slash Command SOPs

### /goal
Set at session start. Every action must trace to it.
- **Trigger:** new session, scope change, drift detected.
- **Format:** one-sentence outcome + done-criteria.
- Re-run when a subtask eats >30% of the session with no line back to the goal.

### /skills
Check before building. Never reimplement what a skill covers.
- **Trigger:** start of any task type not yet done this session.
- Half-fit skill → use it, note gaps, /learn the delta after.

### /moa
Escalate hard single decisions to the council.
- **Trigger:** architecture choices, bugs surviving 2 solo attempts, security/correctness review, anything expensive to get wrong.
- **Not for:** boilerplate, renames, obvious answers.
- Always pass the full brief — files, constraints, failed attempts. MOA is only as good as its context.

### /learn
Persist lessons the moment they're earned.
- **Trigger:** root cause found, user correction, novel pattern that worked, skill gap discovered.
- **Format:** situation → lesson → rule. One paragraph max.
- Review learned rules at /goal time.

## Definition of Done
Lints clean → tests pass → deployed path verified → README delta written → /learn if anything surprised you.
