---
name: session-to-skill
description: Use when the user wants to review the current conversation and turn reusable patterns, procedures, or output formats from it into a Claude Code skill — triggered by requests like "이 세션에서 스킬 뽑아줘", "이거 스킬로 만들 수 있어?", or right after finishing a workflow that feels like it'll come up again.
---

# Session to Skill

## Overview

Looks back at the *current* conversation (already in context — never re-read a transcript file), finds procedures or output formats worth reusing, proposes them to the user, and turns whatever they pick into a properly-structured skill. Registers it locally first, then optionally opens a PR to the shared marketplace at `Jeric1223/my-claude-skills`.

## When to Use

- User explicitly asks to extract/create a skill from what just happened in this session
- Right after a multi-step workflow wraps up and it seems like it'll recur (offer it once, don't insist)
- NOT for a single one-off task with no repeat structure, and NOT for turning project-specific conventions into a skill — those belong in that project's instructions file, not a personal skill

## Process

**1. Scan the conversation for candidates.** Look for: procedures repeated more than once, decisions/rules the user gave that took several messages to pin down, output structures likely to be asked for again. Apply the bar from `superpowers:writing-skills`'s "When to Create a Skill" — not intuitively obvious, reusable across projects, not project-specific.

**2. Draft 2-4 candidates.** For each: a name, one sentence on the reusable pattern, and a rough trigger description. Discard anything that's really just this-project detail wearing a skill's clothes.

**3. Propose via AskUserQuestion.** Multi-select — the user may want more than one, or none of the above (they can always type something else via "Other").

**4. Build each selected skill.** Follow the repo convention exactly (see the marketplace repo's top-level `README.md` "스킬 구조" section, and `CONTRIBUTING.md`): `SKILL.md` (required), `README.md` (required, human-facing), `template.md` (only if output shape must be fixed).

**5. Register it locally.**

Resolve the local skills repo — **never hardcode a path**, try in this order and stop at the first hit:
1. `$CLAUDE_SKILLS_REPO`
2. The repo root that an existing `~/.claude/skills/*` symlink resolves into (`readlink -f`, then walk up to the dir containing `.claude-plugin/marketplace.json`)
3. Ask the user where their skills repo is. If they don't have one, write the skill straight into `~/.claude/skills/<skill-name>/` and skip the symlink — a local-only skill still works.

Then:
- Write the canonical folder into `<repo>/<skill-name>/`
- Symlink it: `ln -s <repo>/<skill-name> ~/.claude/skills/<skill-name>`. If a same-named entry already exists there, **stop and tell the user** instead of overwriting it.
- Add an entry to `<repo>/.claude-plugin/marketplace.json` (`name`, `source`, `description`, `version: "1.0.0"`, `author`) — this file is the only registry; without it the skill isn't installable
- Add a row to `<repo>/README.md`'s 스킬 목록 table

**6. Commit, don't push.** Branch off `main` first (never commit on `main`/`master` directly), commit there — then **stop**. Pushing the user's own repo only happens when they ask.

**7. Offer to share it.** Ask via AskUserQuestion, once, per skill:

> 이 스킬을 공유 마켓플레이스(`Jeric1223/my-claude-skills`)에 PR로 올릴까요?

Options: 올린다 / 로컬에만 둔다. If they decline, stop here — do not ask again.

**8. Sanitize before anything leaves the machine.** The repo is public and this skill's raw material is a *work session*, so scan every file about to be published (`SKILL.md`, `README.md`, `template.md`) and **abort the PR on any hit**, showing the offending lines and asking the user to confirm or edit:

- Cloud account IDs and keys — `\b[0-9]{12}\b`, `\b(AKIA|ASIA)[0-9A-Z]{16}\b`
- Absolute home paths — `/Users/[^/[:space:]]+/`, `/home/[^/[:space:]]+/`
- Private hosts and networks — `\b10\.`, `\b192\.168\.`, `\b172\.(1[6-9]|2[0-9]|3[01])\.`, `\.internal\b`, `\.local\b`
- Credentials — `(password|secret|token|api[_-]?key)\s*[=:]\s*\S`
- **Org-specific terms derived at runtime, not hardcoded**: pull them from the current project's git remotes (`git remote -v`), the org/host segments of any non-GitHub remote, and any company or project names in the user's `CLAUDE.md`. Grep the drafted files for those terms.
- Real infrastructure names carried over from the session — table names, queue names, bucket names, endpoints, ticket IDs

Anything that survives should read as a generic example. If a pattern genuinely needs a concrete value to make sense, replace it with a placeholder (`<account-id>`, `example.com`).

**9. Open the PR.** Requires `gh`:

- `gh auth status` — if `gh` is missing or unauthenticated, **stop before creating any branch**: tell the user to `brew install gh && gh auth login` and re-run. Do not half-build the PR.
- `OWNER=$(gh api user -q .login)`. If `OWNER` is the upstream owner, work on a branch in a clone of the upstream; otherwise `gh repo fork Jeric1223/my-claude-skills --clone=false` and use the fork.
- Clone into a temp dir (`mktemp -d`) — do **not** reuse the user's local repo for this, and never touch its working tree
- Branch `add-<skill-name>`, copy the skill folder in, add the `marketplace.json` entry and the README table row (steps 3 and 4 of `CONTRIBUTING.md`)
- Commit, push to the fork, `gh pr create --repo Jeric1223/my-claude-skills` filling in `.github/PULL_REQUEST_TEMPLATE.md`'s checklist honestly — leave a box unchecked rather than checking something you didn't verify
- Report the PR URL back to the user

**10. Say what's untested.** New skills built this way are a first draft, not verified against RED-GREEN-REFACTOR pressure scenarios. Say so plainly — in the PR body too — rather than implying they've been tested.

## Common Mistakes

- Proposing something so tied to this one project's specifics that it can't actually generalize
- Skipping the AskUserQuestion step and just picking a candidate yourself
- **Opening a PR without running step 8** — a public repo makes leaked internal names permanent
- Hardcoding the local skills repo path instead of resolving it
- Forgetting the `marketplace.json` entry — the skill looks registered but can't be installed
- Pushing the user's own repo without being asked, or committing straight on `main`
- Overwriting an existing `~/.claude/skills/<name>` instead of flagging the collision
- Starting the fork/branch dance before checking that `gh` is authenticated
