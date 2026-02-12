---
allowed-tools: AskUserQuestion, Read, Write, Glob
argument-hint: [filepath or description]
description: In-depth spec interview, then write refined spec to file
---

Input received: $1

First, determine what type of input this is:
- If it looks like a filepath (contains `/` or `\`, ends with `.md`/`.txt`, or starts with `./` or `@`), read the file and use its contents as the initial idea
- Otherwise, treat the input as an inline description of the idea

Then interview me in detail using AskUserQuestion about literally anything: technical implementation, UI & UX, concerns, tradeoffs, edge cases, error handling, scalability, and architectural decisions. Make sure questions are not obvious - dig deep.

Continue interviewing until complete, then:
- If input was a filepath: write the refined spec back to that file
- If input was inline: ask where to save, or create in `./specs/` or `./docs/`
