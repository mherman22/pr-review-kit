# pr-review-kit

One review skill. Two ways to run it.

| | CI sweep | Local deep review |
|---|---|---|
| Invoked by | `@claude review` on a PR | `/pr-review-kit:pr-review <url> --post` |
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

## Prerequisites

Both paths shell out to real tools. Before anything works you need:

| Tool | Why | Check |
|---|---|---|
| Claude Code with plugin support | `/plugin` marketplaces | `claude --version` |
| `gh` CLI, authenticated | every PR read and every comment posted | `gh auth status` |
| `git` | the cached checkouts in §3 of the skill | `git --version` |
| `jq` | JSON handling in the skill's `gh api` calls | `jq --version` |

The CI path additionally needs the [Claude GitHub App](https://github.com/apps/claude)
installed on the org, and a `CLAUDE_CODE_OAUTH_TOKEN` secret (see below).

> `/plugin` and `/pr-review-kit:pr-review` are **Claude Code slash commands**.
> Type them at the Claude Code prompt, not in your shell.
> `zsh: command not found: plugin` means you're in the wrong place.
>
> Every `/plugin ...` command below also has a shell equivalent — `claude plugin
> marketplace add ...`, `claude plugin install ...`, `claude plugin list`. Those
> are the ones to reach for when you're debugging a setup, because they print a
> real success or failure line instead of changing state quietly.

## Setup

### 1. Use it as-is, or fork it

The marketplace lives at `mherman22/pr-review-kit` and is public, so you can
install it directly — skip to §2.

To run your own copy, push this repo somewhere both your laptop and GitHub
Actions can reach:

```bash
gh repo create YOUR-ORG/pr-review-kit --public --source=. --push
```

If you make it private, the Action needs a token with read access to it.

**The marketplace resolves from the pushed default branch, never your working
tree.** If you edit `SKILL.md` or move a file and don't push, the install keeps
serving the old layout and every symptom below looks like a Claude Code bug.
This is the single most common way setup fails for the person who *wrote* the
repo. `git status` before you debug anything else.

Changing the owner or the plugin name means editing **three** files — miss one
and the install silently resolves to somebody else's repo:

- `.claude-plugin/plugin.json` → `repository`
- `.claude-plugin/marketplace.json` → `name` (this is the `@digi-tools` half of
  the install command) and `owner`
- `examples/claude-review.yml` → `plugin_marketplaces` and `plugins`

### 2. Local path

From your shell, so you can see each step succeed or fail:

```bash
claude plugin marketplace add https://github.com/mherman22/pr-review-kit.git
claude plugin install pr-review-kit@digi-tools
```

The equivalent `/plugin marketplace add ...` and `/plugin install ...` work at
the Claude Code prompt too.

**Then restart Claude Code.** A freshly installed plugin is not loaded into the
session that installed it. Skipping this looks exactly like a failed install.

Verify before you try to use it:

```bash
claude plugin details pr-review-kit@digi-tools
```

You want `Skills (1)  pr-review` in the component inventory. If it says
`Skills (0)`, the skill wasn't found — see Troubleshooting.

Then, from any directory:

```
/pr-review-kit:pr-review https://github.com/openmrs/openmrs-module-chartsearchai/pull/85 --post
```

#### The command name is not `/pr-review`

Claude Code namespaces plugin skills as `<plugin>:<skill>`. This plugin is
`pr-review-kit` and its skill is `pr-review`, so the command is the full
`/pr-review-kit:pr-review`. A bare `/pr-review` does not resolve to anything.

Two more names are easy to confuse it with, and you may already have both:

| Name | What it is |
|---|---|
| `/pr-review-kit:pr-review` | this plugin |
| `/pr-review-toolkit:review-pr` | Anthropic's official plugin, a different tool |
| `/pr-reviewer` | a bundled skill, also unrelated |

`claude plugin list` tells you which you actually have.

This works on repos you don't own — it clones to `~/.claude/pr-review-cache/`
on first use and fetches after that. Comments post under your GitHub identity
because the skill uses your `gh` auth.

Drop `--post` to see the findings in your terminal without publishing. Worth
doing on the first few runs.

The skill deliberately does **not** pin a model, so it inherits your session.
Run `/model opus` first if you aren't already there — a local review on Sonnet
is just the CI sweep with extra steps.

#### Arguments

`/pr-review-kit:pr-review <target> [flags]`

The target is a full PR URL, a bare number (resolved against the current repo),
or `OWNER/REPO#NUMBER`.

| Flag | Effect |
|---|---|
| *(none)* | Findings printed to your terminal. Nothing is published. |
| `--post` | Publishes findings as **one** inline review (`event: COMMENT` — never approve or request-changes; that call stays human). |
| `--react` | On a clean pass, adds 👍 to the trigger instead of posting a "looks good" comment. Mainly for CI. |
| `--fix` | Applies fixes to a local checkout after reporting. Needs write access. Don't combine with `--post` unless you mean it. |

#### Updating

The marketplace is a git clone, so edits to `SKILL.md` reach you only after a
refresh:

```
/plugin marketplace update digi-tools
```

To remove it entirely: `/plugin uninstall pr-review-kit@digi-tools`, then
`/plugin marketplace remove digi-tools`.

#### The cache

`~/.claude/pr-review-cache/OWNER__REPO/` accumulates one blobless clone per repo
you review and is never pruned automatically. `rm -rf ~/.claude/pr-review-cache`
is safe at any time — the next review re-clones.

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

#### Environment the skill reads

Two variables in the workflow's `env:` block are load-bearing, not decoration:

| Variable | Why it matters |
|---|---|
| `GH_TOKEN` | Every `gh` call in the skill. Also decides whose name lands on the comments — in CI, the Actions bot. |
| `TRIGGER_COMMENT_ID` | Lets a clean pass 👍 the exact comment that triggered it. Unset, the skill falls back to reacting to the PR itself. |

## What a review looks like

```
PR #85: Add semantic search fallback  (+412/-63 across 9 files)

Adds an embedding-backed fallback when keyword search returns nothing.
Safe to merge once the unbounded retry is addressed.

🔴 Important (1)
  api/src/search/fallback.ts:142 — retry loop has no ceiling; a persistently
  failing embedding service spins until the request times out.

🟡 Nit (1)
  api/src/search/fallback.ts:203 — error swallows the upstream status code.

Verified: 2 confirmed against source, 5 candidates dropped.
```

That last line is the honest signal. `--post` turns the same content into one
review with inline comments; findings on lines the diff doesn't touch move into
the review body under **Additional findings**.

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
├── LICENSE
└── README.md
```

One structural rule: `.claude-plugin/` holds **only** the two manifests.
`skills/` must sit at the plugin root, as a sibling. Move it inside
`.claude-plugin/` and nothing loads, with no error message to tell you why.

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
actors by default. To let a specific bot through, add `allowed_bots` to the
action's `with:` block in `claude-review.yml`:

```yaml
      - uses: anthropics/claude-code-action@v1
        with:
          allowed_bots: "dependabot[bot],renovate[bot]"
```

So `@claude review` from a drive-by commenter on a public repo won't burn a
runner.

On public repos, GitHub withholds secrets from fork-PR runs, so the sweep only
covers PRs from branches in the same repository. Fork PRs need the local path.

## Troubleshooting

**`zsh: command not found: plugin`** — you typed a Claude Code slash command in
your shell. Start `claude`, then type it at that prompt.

**`/plugin marketplace add` fails or finds nothing** — the repo must expose
`.claude-plugin/marketplace.json` at its root on the default branch, and it must
be pushed. A local-only commit is invisible to the marketplace resolver.

**`/pr-review` doesn't exist** — it never did. The command is
`/pr-review-kit:pr-review`. Plugin skills are always namespaced `<plugin>:<skill>`.

**Installed, restarted, still no `/pr-review-kit:pr-review`** — run
`claude plugin details pr-review-kit@digi-tools`. If the inventory says
`Skills (0)`, the plugin loaded but the skill didn't: `skills/pr-review/SKILL.md`
must sit at the plugin root, not under `.claude-plugin/`. That failure is silent
by design; there is no error to grep for.

**Everything installs, but the skill behaves like an older version** — you are
running the pushed default branch, not your working tree. Push, then
`claude plugin marketplace update digi-tools`, then restart.

**The Action runs but posts nothing** — check, in order: the `prompt:` uses the
namespaced `/pr-review-kit:pr-review` (a bare `/pr-review` burns a runner and
does nothing); the `if:` gate matched the comment body; `pull-requests: write`
and `issues: write` are both present in `permissions:`; `CLAUDE_CODE_OAUTH_TOKEN`
is set and not expired. A clean review with `--react` posts *only* a 👍, which is
the intended quiet path.

**Review posted as the wrong identity** — comments follow whoever owns
`GH_TOKEN`. In CI that's `secrets.GITHUB_TOKEN` (the Actions bot); locally it's
your `gh auth` login.

## Costs

The CI sweep bills against the subscription behind `CLAUDE_CODE_OAUTH_TOKEN`,
plus GitHub Actions minutes. Keep it on the comment trigger rather than
`pull_request: [opened]` until you know what a sweep costs you on a real repo.
`--max-turns 60` is the backstop against a runaway run on a large diff.

Anthropic also sells a managed Code Review product that does the multi-agent
version of this with inline severity annotations. It's Team/Enterprise only and
runs roughly $15–25 a review, which is exactly the cost problem this repo
exists to route around.

## License

MIT — see [LICENSE](LICENSE).
