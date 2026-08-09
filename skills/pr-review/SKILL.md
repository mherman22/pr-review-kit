---
name: pr-review
description: Review a GitHub pull request for correctness bugs, security issues, and regressions. Takes a PR URL, number, or owner/repo#number. Use when asked to review a PR, check a pull request, or when invoked as part of an automated review pipeline.
argument-hint: <pr-url-or-number> [--post] [--react] [--fix]
allowed-tools: Bash(gh:*), Bash(git:*), Bash(mkdir:*), Bash(cd:*), Bash(jq:*), Bash(cat:*), Read, Grep, Glob, Write
---

# PR Review

Review a GitHub pull request. Arguments: `$ARGUMENTS`

This skill is the single source of truth for review logic. It runs unchanged in
two contexts — a CI runner on a cheap model, and a local session on a strong
one. **Do not branch your review standards on which context you're in.** The
only thing the context changes is how findings get published.

## 1. Parse arguments

From `$ARGUMENTS` extract:

- **Target** — a URL like `https://github.com/OWNER/REPO/pull/NUMBER`, a bare
  number (use the current repo), or `OWNER/REPO#NUMBER`. Derive `OWNER`,
  `REPO`, `NUMBER`. Pass `--repo OWNER/REPO` on every `gh` call so this works
  from any directory.
- **`--post`** — publish findings as inline PR comments. Without it, report to
  the terminal and publish nothing.
- **`--react`** — when the review finds nothing worth reporting, add a 👍
  reaction to the trigger instead of posting a "looks good" comment. See §7.
- **`--fix`** — apply fixes to a local checkout after reporting. Requires write
  access. Never combine with `--post` unless explicitly asked.

If you can't parse a target, stop and say so. Don't guess a PR number.

## 2. Load the PR

```
gh pr view NUMBER --repo OWNER/REPO --json title,body,author,isDraft,baseRefName,headRefName,headRefOid,additions,deletions,changedFiles,labels,comments,reviews
gh pr diff NUMBER --repo OWNER/REPO
```

**Stop without reviewing** — and say which reason applies — if:

- A review from this same identity already covers the current head SHA
- The PR is a pure dependency bump, generated-file update, or version bump
- The diff is entirely lockfiles, snapshots, or vendored code

Review Claude-authored PRs normally. They are not exempt.

## 3. Build real context

A diff alone hides most real bugs. You need the code around it.

- If the working directory is already `OWNER/REPO` with a clean tree:
  `git fetch origin && gh pr checkout NUMBER`
- If the tree is dirty, leave it alone. Read files at the PR head instead:
  `gh api repos/OWNER/REPO/contents/PATH?ref=HEADSHA --jq .content | base64 -d`
- Otherwise cache a checkout:
  ```
  mkdir -p ~/.claude/pr-review-cache
  gh repo clone OWNER/REPO ~/.claude/pr-review-cache/OWNER__REPO -- --filter=blob:none
  cd ~/.claude/pr-review-cache/OWNER__REPO && git fetch origin && gh pr checkout NUMBER --force
  ```

Read `CLAUDE.md` at the repo root and in any directory containing changed
files. Its conventions are review criteria.

Read every changed file **in full**, not just the hunks. Then read the callers
and callees of anything touched. The bugs that matter are the ones the diff
doesn't show.

## 4. Review

In priority order:

1. **Logic errors** — off-by-one, inverted conditions, wrong operator, bad
   state transitions, misuse of an API's actual contract
2. **Security** — injection, authz gaps, unscoped queries, secrets or PII in
   logs and errors, unsafe deserialization
3. **Edge cases** — null/empty/zero, races, partial failure, unbounded growth,
   error paths that swallow or misreport
4. **Regressions** — callers this breaks, behavior changes no test covers,
   migrations that aren't backward compatible
5. **Convention violations** — only where `CLAUDE.md` or the surrounding code
   sets a clear rule

Do **not** report: formatting, naming preferences, refactors that fix no defect,
anything CI already enforces, or pre-existing issues this PR didn't touch —
unless the PR makes one materially worse, in which case say so explicitly.

## 5. Verify every finding before it counts

This step is what makes a cheap model's review usable and an expensive model's
review trustworthy. For each candidate:

- Cite the exact `file:line` of the problem
- Trace whether the bad path is actually reachable
- Check whether existing code already guards it
- If the claim depends on what a function does, **read that function**. Never
  infer behavior from a name.

**Drop anything you cannot confirm this way.** A confident wrong finding costs
the author more than a missed nit does. Report how many candidates you dropped
in the terminal (§6). That number is the honest signal of how hard you looked,
but it belongs to you, not to the PR author.

## 6. Report to the terminal

```
PR #NUMBER: <title>  (+A/-D across F files)

<one line: what this PR does, and whether it looks safe to merge>

Bugs (N)
  path/to/file.ts:142  <what breaks, and when>

Nits (N)
  path/to/file.ts:203  <issue>

Verified <X>, dropped <Y> unconfirmed.
```

If nothing survived verification, say so in one line. Never manufacture a
finding to justify the run.

The dropped count stays in the terminal. It tells *you* how hard the review
looked. It is noise to the PR author, so it never goes in a posted comment.

## 7. Publish

### Clean review

If there are no findings:

- **With `--react`**: add 👍 to the trigger and post nothing.
  - If `TRIGGER_COMMENT_ID` is set in the environment, react to that comment:
    ```
    gh api repos/OWNER/REPO/issues/comments/$TRIGGER_COMMENT_ID/reactions \
      --method POST -f content='+1'
    ```
  - Otherwise react to the PR itself:
    ```
    gh api repos/OWNER/REPO/issues/NUMBER/reactions --method POST -f content='+1'
    ```
- **With `--post` but no `--react`**: post a one-line summary comment.
- **With neither**: terminal output only.

The 👍 is load-bearing. It means *the review ran and found nothing*, which is a
different statement from *the review didn't run*. Never skip it on a clean pass.

### How to write the comments

You are writing to a tired engineer who has eight other tabs open. They will
read your first sentence and skim the rest. Write for that person.

**Hard limits.** An inline comment is at most three sentences. Sentence one
names the problem and must make sense alone. Sentence two gives the evidence
with a `file:line`. Sentence three, if it exists, is the fix. Cut anything that
survives after that. The review summary is at most four sentences.

**Never use:**

- Emoji, anywhere.
- Bold severity labels. Start with a plain `Bug:` or `Nit:` and nothing else.
- Em dashes. Use a period or a comma.
- Preamble: "It's worth spelling out that", "The headline issue is", "Note that".
- Transition words you'd never say aloud: "Additionally", "Moreover", "Furthermore".
- The "not just X, but Y" construction, and lists of exactly three things where
  two would do.
- Restating what the PR does before saying what's wrong with it.

**Always:**

- Name the symptom before the mechanism. "Every save returns 400" lands; "the
  argument resolver runs bean validation before the controller body" does not,
  until they know why they should care.
- Say the concrete consequence. Which user, doing what, sees what break.
- Use backticks for identifiers, not bold.
- Give the fix as a short clause, not a paragraph of options.

Compare. Too long:

> 🔴 **Important** — `@NotNull @Size(min = 1)` makes the auto-generation feature
> unreachable and breaks the existing catalog form.
>
> `InventoryItemRestController.create()` and `update()` both declare `@Valid
> @RequestBody InventoryItem item`. `@EnableWebMvc` is on `AppConfig:55`,
> hibernate-validator 8.0.2 is a compile dependency, and `ControllerSetup:87`
> handles `MethodArgumentNotValidException` with a 400, so validation fires in
> the argument resolver, before the method body. [...]

What to write instead:

> Bug: every Add and Edit in the catalog form will 400 after this. `code` is
> `@NotNull`, but `InventoryItemForm.jsx` never sends it, and `@Valid` rejects
> the payload before `resolveCode()` can fill it in. Drop the annotation and let
> `nullable = false` enforce it.

Same finding, same evidence, one third the length.

### Findings to publish

With `--post`, submit **one review**, not N loose comments. Write the payload
to a temp file:

```json
{
  "event": "COMMENT",
  "body": "<summary: what this does, whether it's safe to merge, the one thing that matters most>",
  "comments": [
    { "path": "src/auth/session.ts", "line": 142, "side": "RIGHT",
      "body": "Bug: sessions survive logout. `revoke()` clears the cookie but never deletes the row, so a replayed token still resolves at session.ts:88. Delete the record here." }
  ]
}
```

```
gh api repos/OWNER/REPO/pulls/NUMBER/reviews --method POST --input /tmp/pr-review-NUMBER.json
```

Rules:

- Always `event: "COMMENT"`. Never `APPROVE` or `REQUEST_CHANGES`. That call
  belongs to a human.
- `line` must exist in the diff on the `RIGHT` side. A finding about an
  untouched line goes at the end of the review `body` under a plain
  `Also:` line, one sentence each, not inline.
- No finding counts, no verification tally, and no list of what you dropped.
  That is terminal output. The author only needs the findings themselves.
- If the API rejects the payload, report the error *and* the full findings to
  the terminal. Never retry with a mangled payload.

Then print the review URL.

## Note on identity

Comments are attributed to whoever owns the token in `GH_TOKEN`. In CI that's a
bot; locally it's you. The review content is identical either way — only the
name on the comment differs.
