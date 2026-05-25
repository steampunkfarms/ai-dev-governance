# Contributing

This repo documents a **governance model**, not a tool — so the most valuable contributions are *lessons*, not features. If you've hit a wall this playbook doesn't cover, or found a cleaner way to handle one it does, we'd love to fold it in.

## What makes a good contribution

- **A new wall.** A failure mode of AI-assisted, multi-repo development that we haven't documented, plus the discipline that prevents it.
- **A clearer explanation.** Same lesson, fewer words, better example.
- **A new facet deep-dive.** An isolated treatment of one part of the model (see below).
- **A correction.** Something here that's wrong, dated, or doesn't generalize.

## How to propose a change

1. **Open an issue first** for anything non-trivial. Describe the *failure mode* and how your approach prevents it — that framing is the whole point of the repo.
2. Fork, branch (`git checkout -b lesson/<short-name>`), and keep changes small and focused.
3. Open a PR that references the issue.

## Adding a facet deep-dive

Copy [`docs/_deep-dive-template.md`](docs/_deep-dive-template.md) and keep the shape: **the wall → the rule → how it works → a worked example → anti-patterns → adapt it to your setup.** Link it into the "Deep dives" table in the README.

## Style

- **Stay actor-agnostic.** Write in terms of the *Strategist* and *Executor* roles. Real tools (Claude Code, Codex, Cursor, …) appear only as examples, never as requirements.
- **De-identify examples.** No real company names, repo names, client data, or internal jargon. A generalized story teaches just as well.
- **Lessons first.** Lead with the mistake and the cost, then the fix.
- **Keep it tight.** If a paragraph doesn't change what someone *does*, cut it.

Thanks for helping other developers skip the walls you already climbed.
