---
name: upstream-pr
description: Pre-submission gate for any pull request from this fork (sparkmastergrape/ans) to upstream agentnameservice/ans. Use before opening an upstream PR, before pushing a round of review fixes to one, and whenever asked to "send this upstream", "open a PR to agentnameservice", or check that a contribution is ready.
---

# Upstream contribution gate

A PR to `agentnameservice/ans` spends a maintainer's time. It should
arrive finished: the reviewer confirms the work, they don't find its
bugs. Every item below must hold before the PR is opened and again
before each review-fix push. If one fails, fix it or don't submit.

## Why this exists

PR #131 (`ans-verify -fqdn`) went up with green lint, race, and
coverage gates. The maintainer then ran the binary against fixtures and
found two blocking security bugs in an afternoon:

1. A hostname whose `_ans-badge` pointed at **another host's** valid
   registration printed VERIFIED and exited 0. Every signature checked
   out. Nothing tied the signed registration back to the name asked
   about.
2. A trailing-dot (absolute) FQDN was re-queried as a relative name,
   so a DNS search domain could answer for it.

Both were cheap to find. Unit tests checked the code we wrote, but
nothing checked whether the claim the tool makes holds against hostile
input. The gate below is aimed at that gap.

## 1. Scope

- An upstream issue exists and the PR closes it (`Fixes #N`). The PR
  does what the issue asks: nothing missing and nothing extra. Name
  any out-of-scope items in the description, as #131 did.
- Large or design-level changes get agreement on the issue first.
- The branch is cut from **current upstream `main`**, not this fork's
  `main` (which lags). Check with
  `git log --oneline upstream/main..HEAD`. It must list only this
  PR's commits.
- One concern per PR. Dependency bumps, drive-by refactors, and
  unrelated fixes go in their own PRs.

## 2. Break it before the reviewer does

Write down, in one sentence, the **claim** the change makes. Examples:
"exit 0 means this FQDN is registered in ANS", "the TL rejects a V1
body on the V2 route". Then attack it:

- **Wrong-binding inputs.** Is a valid proof for X accepted as proof
  for Y? Check names, IDs, hosts, keys, and log URLs taken from
  untrusted sources such as DNS, badges, request bodies, and headers.
- **Normalization.** Try trailing dots, case, whitespace, IDN/punycode,
  default ports, trailing slashes, and duplicate or conflicting flags.
  Is the value that gets compared the same value that gets used?
- **Trust provenance.** For every key, URL, or ID, ask who supplied it.
  If the thing being verified also supplies the key, say so in the
  output and docs, or require the caller to pin it.
- **Environment leakage.** Check what resolver search lists, proxies,
  env vars, and system CA or trust stores can change.
- **Failure paths.** Every failure must exit non-zero with a specific
  message. "Skipped" must never be reported as "passed".

Each attack that the code survives becomes a regression test that
fails on the pre-fix code. Confirm that it fails before the fix.

## 3. Run the real thing

Unit tests are necessary but not sufficient. Build the binary
(`make build`) and drive the changed behavior end to end against
fixtures or the `scripts/demo/` stack. Run the hostile cases from §2,
not just the happy path. Paste the commands and their output, including
at least one rejected case, into the PR description under
**Testing**.

## 4. Repository bar (CLAUDE.md)

- `make check` passes: fmt, vet, golangci-lint, and the 90% coverage
  gate. Also run `make test-race`.
- `internal/domain` stays at 100%. Any `internal/crypto` exception has
  a SAFETY or NOTE comment.
- Logging follows CLAUDE.md. Use an injected `zerolog` logger with a
  `component` field, and never log secrets. CLI output in `cmd/` is
  user-facing and stays explicit about what was and wasn't verified.
- No TODOs, no placeholder routes, and no partially implemented flags.
- If the change touches a handler, DTO, event envelope, or receipt
  shape, paste the shape diff against `spec/` into the description,
  with every differing field called out.
- README and docs are updated for any new flag or behavior.

## 5. Self-review

Before opening the PR, run `/code-review high` on the diff against
`upstream/main`. Then do one pass reading the diff as the maintainer
would, asking what would make you request changes. Resolve every
finding or explain it in the description.

## 6. Commits and description

- Every commit is DCO `Signed-off-by:` by the human submitter and
  GPG-signed. AI tools never add `Signed-off-by:` lines, and commits
  carry no `Co-Authored-By:` trailer naming an AI.
- Commits are Conventional Commits, for example
  `feat(ans-verify): …` or `fix(dns): …`.
- The description includes: what changed and why; scope and non-goals;
  the §2 claim and the attacks tried; the §3 testing transcript; the
  shape diff, if applicable; and an **AI assistance** section naming
  the tool and model (CONTRIBUTING.md).

## 7. During review

- Answer every review thread. When a thread is fixed, name the commit
  and the test that covers it.
- Treat a reviewer's finding as a **class** of bug. Re-run §2 for the
  whole class across the PR, not just the line they flagged.
- Push fixes as new commits on the PR branch. Never force-push after
  review has started.
- After pushing, re-request review from whoever requested changes.
  Don't leave the PR waiting silently.
- Don't post upstream (PRs, comments, re-requests) without the user's
  go-ahead in this session.
