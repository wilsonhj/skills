# Skills audit: what to bring into my Claude Code setup

Audit of this fork (`wilsonhj/skills`) as of 2026-09-25, to decide which skills to install into a personal Claude Code setup.

## Provenance and safety

- The fork is **identical to upstream** `mattpocock/skills` `main` (0 commits ahead, 0 behind). Nothing here is fork-specific yet.
- Almost every skill is plain Markdown prompting. The only executables are:
  - `scripts/link-skills.sh`: symlinks skills into `~/.claude/skills` and `~/.agents/skills`. **Caveat:** if a real (non-symlink) directory with the same name already exists there, it runs `rm -rf` on it before linking. It also links everything in `in-progress/`.
  - `skills/engineering/wizard/template.sh`: interactive bash template; writes `.env` and calls `gh secret set` / `gh variable set` only when the human runs it.
  - `skills/engineering/diagnosing-bugs/scripts/hitl-loop.template.sh`: a human-in-the-loop repro template.
  - `skills/misc/git-guardrails-claude-code/scripts/block-dangerous-git.sh`: a PreToolUse hook that greps the Bash command for patterns like `git push` and `reset --hard`.
- No network calls, credential reads, or obfuscated content found. Safe to adopt.

## How the skills fit together

The skills form one opinionated pipeline plus standalone tools:

```
grill-with-docs -> (prototype) -> to-spec -> to-tickets -> implement (tdd + code-review)
on-ramps: triage, diagnosing-bugs, wayfinder
```

The pipeline skills (`to-spec`, `to-tickets`, `triage`, `wayfinder`, `code-review`) expect per-repo config from `/setup-matt-pocock-skills`, which writes `docs/agents/issue-tracker.md`, triage labels, and an `## Agent skills` section into `CLAUDE.md`/`AGENTS.md`. They use the `gh` (or `glab`) CLI for issue trackers, or local files under `.scratch/`.

## Recommendation

### Tier 1: adopt now (standalone, no per-repo setup)

| Skill | Why | Invocation |
| --- | --- | --- |
| `grilling` + `grill-me` | The repo's best idea: structured interview in rounds with a recommended answer per question. Use before any non-trivial change. `grill-me` is a one-line wrapper around `grilling`, so install both. | model / user |
| `diagnosing-bugs` | Disciplined debug loop: refuses to theorise until there is one command that goes red on the bug, then fixes with a regression test. | model |
| `tdd` + `codebase-design` | Good red-green rules (vertical slices, no tautological tests, test at agreed seams). `tdd` calls `codebase-design` for vocabulary, so take both. | model |
| `resolving-merge-conflicts` | Resolves by tracing each side's intent to its commit/PR. Note it never aborts and commits when done. | model |
| `prototype` | Throwaway logic (single HTML file) or UI-variant prototypes to settle a design question. | model |
| `handoff` | Writes a portable handoff doc to the OS temp dir for a fresh session. | user |
| `research` | Delegates primary-source research to a background agent and writes a cited Markdown file. | model |
| `writing-for-agents` | Reference for writing skills and `CLAUDE.md`. Worth having if you author your own skills. | model |

### Tier 2: adopt if you want the full spec-to-ship workflow

Take these as a set. Run `/setup-matt-pocock-skills` once per repo first.

`setup-matt-pocock-skills`, `grill-with-docs`, `domain-modeling`, `to-spec`, `to-tickets`, `implement`, `code-review`, `triage`, `wayfinder`, `improve-codebase-architecture`, `ask-matt` (router that explains the flow), `wait-what` (re-explain using `CONTEXT.md` terms).

Trade-offs to know about:

- They create files in your repos (`CONTEXT.md`, `docs/adr/`, `docs/agents/`, `.scratch/`) and edit `CLAUDE.md`.
- `triage`, `to-spec`, and `to-tickets` apply labels and open issues on your tracker.
- **Name collision:** `code-review` shadows Claude Code's built-in `/code-review` when installed as a loose skill in `~/.claude/skills`. `implement` calls `/code-review` by name, so the two get confused. Installing as a plugin namespaces it (`mattpocock-skills:code-review`), which avoids the clash. Otherwise rename it (for example `two-axis-review`) and update `implement`.

### Tier 3: situational, install only if the use case applies

- `teach`: multi-session learning workspace.
- `to-questionnaire`: writes a questionnaire for someone else to fill in.
- `wizard`: generates an interactive bash script for human-only steps (dashboards, secrets). Handy for infra setup.
- `in-progress/retro`: suggests improvements to your agent environment after a session. Beta, but useful.
- `in-progress/pr`: PR body template. Beta.

### Skip

- `misc/migrate-to-shoehorn`, `misc/scaffold-exercises`: specific to the author's TypeScript courses.
- `misc/setup-pre-commit`: Husky + lint-staged only; fine for JS repos, otherwise irrelevant.
- `misc/git-guardrails-claude-code`: the idea is sound, but the hook matches substrings naively (blocks `echo "git push"`, misses `git -C dir push`). Use `permissions.deny` rules in `~/.claude/settings.json` instead.
- `in-progress/implement-spec`: superseded by `implement`.
- `in-progress/claude-handoff`: depends on `claude --bg`; use `handoff` instead.
- `in-progress/writing-*`, `in-progress/loop-me`, `in-progress/setup-ts-deep-modules`: prose-writing and TS-specific experiments.

## Context cost

Only model-invoked skills put their description in every session's context. Tier 1 + Tier 2 together add about 11 model-invoked descriptions (a few hundred tokens). User-invoked skills (`disable-model-invocation: true`) cost nothing until you type them.

## How to install

Pick one route. Do not combine them, or you get every skill twice.

**A. Whole promoted set as a plugin (simplest, tracks upstream):**

```bash
claude plugins install mattpocock-skills
```

Gets all 25 promoted skills (Tier 1 + Tier 2 + `teach`, `to-questionnaire`, `wizard`), namespaced, auto-updated from upstream.

**B. Your fork as the plugin source (you control the list):** trim `.claude-plugin/plugin.json`'s `skills` array in this fork to the ones you want, then:

```
/plugin marketplace add wilsonhj/skills
/plugin install mattpocock-skills@mattpocock
```

Run `claude plugin validate . --strict` after editing the manifest. Pull from upstream periodically to get fixes.

**C. Cherry-pick as loose skills (editable, no plugin):** clone the fork locally and symlink only what you want, rather than running `scripts/link-skills.sh` (which links everything outside `misc/` and `deprecated/` and deletes same-named directories):

```bash
cd ~/code/skills   # your local clone
mkdir -p ~/.claude/skills
for s in productivity/grilling productivity/grill-me productivity/handoff \
         productivity/writing-for-agents engineering/diagnosing-bugs \
         engineering/tdd engineering/codebase-design \
         engineering/resolving-merge-conflicts engineering/prototype \
         engineering/research; do
  ln -sfn "$PWD/skills/$s" ~/.claude/skills/"$(basename "$s")"
done
```

Add Tier 2 skills the same way if you want the workflow, and rename `code-review` first to avoid the built-in clash.
