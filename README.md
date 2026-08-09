# pr-review-kit

One review skill. Two ways to run it.

| | CI sweep | Local deep review |
|---|---|---|
| Invoked by | `@claude review` on a PR | `/pr-review <url> --post` |
| Model | Sonnet — cheap enough to run on everything | Opus — for the PRs that matter |
| Runs on | GitHub Actions runner | your laptop |
| Comments appear as | the Actions bot | **you** |
| Clean result | 👍 on the trigger comment | one-line summary |

The point is that `skills/pr-review/SKILL.md` is the **only** copy of the review
logic. Both paths load that same file. Tighten the verification bar once and
both the cheap sweep and the expensive second opinion get stricter together.

## Why the two-tier split

A cheap model reviewing every PR catches the boring stuff at a price you can
afford to pay on all of them. It is not good enough to be the last word on a
migration or an auth change. So the sweep runs on everything, and you escalate
by hand — same skill, better model — when a PR deserves it.

The 👍 exists so a clean sweep is distinguishable from a sweep that never ran.
An empty PR thread is ambiguous; a thumbs-up is not.

## Setup

### 1. Publish this repo

Push it somewhere both your laptop and GitHub Actions can reach:

```bash
gh repo create DIGI-UW/pr-review-kit --public --source=. --push
```

It has to be reachable by the runner. If you make it private, the Action needs
a token with read access to it.

If you use a different owner or name, update `repository` in
`.claude-plugin/plugin.json` and the `plugin_marketplaces` URL in
`examples/claude-review.yml`.

### 2. Local path

```
/plugin marketplace add https://github.com/DIGI-UW/pr-review-kit.git
/plugin install pr-review-kit@digi-tools
```

Then, from any directory:

```
/pr-review https://github.com/openmrs/openmrs-module-chartsearchai/pull/85 --post
```

This works on repos you don't own — it clones to `~/.claude/pr-review-cache/`
on first use and fetches after that. Comments post under your GitHub identity
because the skill uses your `gh` auth.

Drop `--post` to see the findings in your terminal without publishing. Worth
doing on the first few runs.

The skill deliberately does **not** pin a model, so it inherits your session.
Run `/model opus` first if you aren't already there — a local review on Sonnet
is just the CI sweep with extra steps.

### 3. CI path

Per repo you want swept:

```bash
mkdir -p .github/workflows
cp examples/claude-review.yml .github/workflows/
```

Add a `CLAUDE_CODE_OAUTH_TOKEN` secret. Generate one with `claude setup-token`
locally — it authenticates against your Claude subscription instead of billing
API usage. Store it as an org-level Actions secret so each repo doesn't need
its own copy.

Install the [Claude GitHub App](https://github.com/apps/claude) on the org.

Then comment `@claude review` on any PR.

Note that an OAuth token is tied to the subscription of whoever ran
`claude setup-token`. For a shared org secret an API key from the
[Console](https://console.anthropic.com) is the more durable choice — swap
`claude_code_oauth_token` for `anthropic_api_key` in the workflow if you go
that way.

## Layout

```
pr-review-kit/
├── .claude-plugin/
│   ├── plugin.json          # plugin manifest
│   └── marketplace.json     # marketplace registry
├── skills/
│   └── pr-review/
│       └── SKILL.md         # ← the only copy of the review logic
├── examples/
│   └── claude-review.yml    # copy into each swept repo
└── README.md
```

One structural rule: only `plugin.json` lives in `.claude-plugin/`. `skills/`
must sit at the plugin root. Move it inside `.claude-plugin/` and nothing
loads, with no error message to tell you why.

## Tuning

Everything worth changing is in `SKILL.md`:

- **§4** — what gets reported, and the explicit do-not-report list
- **§5** — the verification bar. This is the highest-leverage section. It's
  what stops the cheap model from filling PRs with plausible nonsense.
- **§7** — publishing behavior, including the 👍

Per-repo rules belong in that repo's `CLAUDE.md`; the skill reads it at the
root and in every directory containing a changed file.

## Who can trigger it

The action rejects triggers from users without write access, and rejects bot
actors unless you list them in `allowed_bots`. So `@claude review` from a
drive-by commenter on a public repo won't burn a runner.

On public repos, GitHub withholds secrets from fork-PR runs, so the sweep only
covers PRs from branches in the same repository. Fork PRs need the local path.

## Costs

The CI sweep bills against the subscription behind `CLAUDE_CODE_OAUTH_TOKEN`,
plus GitHub Actions minutes. Keep it on the comment trigger rather than
`pull_request: [opened]` until you know what a sweep costs you on a real repo.
`--max-turns 60` is the backstop against a runaway run on a large diff.

Anthropic also sells a managed Code Review product that does the multi-agent
version of this with inline severity annotations. It's Team/Enterprise only and
runs roughly $15–25 a review, which is exactly the cost problem this repo
exists to route around.
