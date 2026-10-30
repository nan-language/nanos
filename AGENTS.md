# AGENTS.md

## Project

NaN OS is an operating system built for AI agents from the root, from the boot loader and kernel to drivers and user interfaces, rather than AI added on top of a system made for humans. It is written in NaN and targets x86_64.

## Architecture

The repository root is a NaN workspace: each directory holding a `package.nan` is a package, a program when its manifest names an `entry`. `loader` is the boot loader, a UEFI application (`x86_64-uefi`), and `kernel` the kernel (`x86_64-bare`). `nan build` writes each program's image into `.nan/images/<target>/`, and `nan run loader` boots the loader from a FAT volume that also holds every other UEFI and bare image, `kernel.elf` included. Tests are `test` items beside the code they exercise; each of their runs on `x86_64-uefi` or `x86_64-bare` boots QEMU.

Only what an agent needs to place a change correctly. The why is in ADRs.

## Layout

| Path | Holds |
| --- | --- |
| `loader/` | The `loader` package |
| `kernel/` | The `kernel` package |
| `tools/` | The `install` and `sandbox` scripts |
| `docs/` | Documentation: the specification in `docs/spec/`, ADRs in `docs/decisions/` |

One row per top-level directory; deeper only where the tree alone would not tell an agent where a change belongs.

## Commands

| Command | Purpose |
| --- | --- |
| `tools/install` | Install the NaN toolchain release the project pins |
| `nan check [<scope>]` | Report errors and incompleteness |
| `nan fmt [<scope>]` | Rewrite files into canonical format |
| `nan test [<scope>]` | Run the tests of a package, module, item or single test |
| `nan build [<scope>]` | Build the images of the scope's programs, applying the final-build rules |
| `tools/sandbox start [<instance>]` | Boot NaN OS under QEMU from a fresh copy of the working tree, its serial console on standard output |
| `tools/sandbox reset [<instance>]` | Delete a sandbox instance |

Run `nan` from the repository root. A scope is a package, module or item path, a `.nan` file or a `file:line`; omitted at the root, it is the whole workspace.

Design each command for who runs it: agent-facing commands run without interaction, accept a scope where one applies and print only what an agent needs. Agent-facing commands are listed here, human-facing ones in README.md, shared ones in both.

## NaN

NaN OS is written in NaN. Read `nan primer` before writing or reviewing NaN code; `nan primer <topic>` prints one of the topics `low-level`, `kernel`, `concurrency`, `packages`, `contracts` and `components`, and `nan primer tools` the commands and their output. `nan show <path> --api` prints the head, contracts and digest of any item, std included. The specification is at https://nanlang.org/spec/.

Whenever you hit a defect, a wrong or unclear diagnostic or a missing feature in NaN or its toolchain, open an issue with `gh issue create --repo nan-language/nan`, unless `gh issue list --repo nan-language/nan --search "<terms>"` finds it already reported. Give the smallest reproduction, the command and its output, and link the issue in your report.

## Economy

**Context.** Everything committed to this repository is read by every agent that works here later: code, comments, docs, tests, ADRs and this file. Every token that carries no necessary information costs money and dilutes attention on the information that matters. Each kind of knowledge has one home: the **why** in ADRs, the **what** in documentation, the **how** in the code.

**Time.** Every check, run and read costs time. Do each once, at the narrowest scope that proves what you need, and skip anything your change cannot affect.

## Git

Never run a command that alters git history or the remote (commit, amend, rebase, merge, reset, cherry-pick, push, tag, branch deletion) without explicit approval from the user for that specific command. Approval covers one action and never carries over to the next.

## Code

- Follow industry best practice. No over-engineering, no quick fixes.
- The project will grow a lot. Structure it so the next change is easy, not so future features are pre-built. Be smart, not exhaustive.
- Keep the tree, the files and the code organised and clear without being verbose. Code explains the how by itself.
- When the change you are making tips the existing structure into being less maintainable, refactoring is part of that change. Do it now.
- Leave nothing unused: no dead code, no dormant logic, no remnant of a replaced approach. The codebase reflects exactly what it does today.

## Environment

Use and maintain the sandbox defined in this repository to run, observe and test everything, dependencies and data included. Sessions run in parallel: each one, the user's included, starts an isolated instance, so the user can always run the project while agents run theirs. Keep the sandbox working and fast to start and reset.

## Verification

Never guess: run the code in the sandbox and observe. Verify the full behaviour of what you change, including its callers. Never deliver untested work.

## Tests

Write tests that block regressions. Each test costs context, maintenance and pipeline time, so every test must protect a behaviour that matters and run fast; nothing for the sake of coverage.

Lay out tests so one module's or feature's tests run alone through a scoped command. Run only the tests for your change; the full suite belongs to the pipeline.

## Comments

A comment is re-read by every agent that passes through its file and costs context each time. Once the why, the what and the how have their homes, very few comments are legitimate.

- Comment only what cannot be understood from the code itself.
- Describe the code as it is now: no former behaviour, no plans.
- Keep comments within the scope where they appear: a comment inside a function describes only that function; a file-level comment describes only that file. Never reference code or behaviour outside that scope, including as a reason for the local implementation.
- What is in your context is not what is in the code. Never record a decision abandoned along the way, an alternative you considered, or what your change replaced: after replacing A with B, do not comment that A is not used.
- Create, update and delete comments in the same change that alters the code they describe.
- If you hesitate over a comment, leave it out.

## Documentation

Documentation explains the what, in plain human English, structured for a reader. Most work does not need documentation. Add documentation only when the information is important for future readers to understand or use the project; treat every addition, even a word, as a deliberate choice. Create a new document only when that information cannot be covered clearly in an existing one.

- Describe the present state: no history, no plans.
- What is in your context is not what the reader needs. Never document a decision abandoned along the way, an alternative you considered, or what your change replaced: after replacing A with B, document B and never mention A.
- Keep all documentation at one uniform level of detail. What you have just built looms large in your context; do not detail it more than the rest.
- Create, update and delete documentation in the same change that alters what it describes.
- If you hesitate over a sentence, leave it out.

Every document except ADRs is listed here. Read one only when the task needs it.

| Document | Covers |
| --- | --- |
| `docs/spec/` | The NaN OS specification, one chapter per mechanism; `00-charter.md` maps the chapters |
| `docs/sandbox.md` | The sandbox: what an instance runs and how to observe it |

## ADR

Record every architecturally significant decision (structure, dependencies, interfaces, non-functional trade-offs) in `docs/decisions/NNNN-short-title.md`, NNNN incrementing. An ADR holds the why: exactly what is needed to understand the decision, nothing more.

```markdown
# NNNN. Title

Status: accepted | superseded by NNNN

## Context
## Decision
## Consequences
```

Never edit an accepted ADR beyond its status line; a new ADR supersedes it.

## Skills

Create a skill at `.agents/skills/<name>/SKILL.md` only for a task agents will repeat frequently over the life of the repository and that a written procedure makes faster or safer. Keep it current; delete it when the task disappears.

## AGENTS.md and README.md

You may edit this file only to: refresh the project description when it is no longer accurate (never to add detail); update Architecture and Layout when a change alters them; add, update or remove commands; add, update or remove a documentation entry. Everything else here is owned by the user.

README.md is the landing page for humans: description, installation, usage, commands, in clear sections. Keep it accurate and strictly minimal.
