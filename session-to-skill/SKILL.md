---
name: session-to-skill
description: Use when the user wants to review the current conversation and turn reusable patterns, procedures, or output formats from it into a Claude Code skill — triggered by requests like "이 세션에서 스킬 뽑아줘", "이거 스킬로 만들 수 있어?", or right after finishing a workflow that feels like it'll come up again.
---

# Session to Skill

## Overview

Looks back at the *current* conversation (already in context — never re-read a transcript file), finds procedures or output formats worth reusing, proposes them to the user, and turns whatever they pick into a properly-structured skill registered in `my-claude-skills`.

## When to Use

- User explicitly asks to extract/create a skill from what just happened in this session
- Right after a multi-step workflow wraps up and it seems like it'll recur (offer it once, don't insist)
- NOT for a single one-off task with no repeat structure, and NOT for turning project-specific conventions into a skill — those belong in that project's instructions file, not a personal skill

## Process

**1. Scan the conversation for candidates.** Look for: procedures repeated more than once, decisions/rules the user gave that took several messages to pin down, output structures likely to be asked for again. Apply the bar from `superpowers:writing-skills`'s "When to Create a Skill" — not intuitively obvious, reusable across projects, not project-specific.

**2. Draft 2-4 candidates.** For each: a name, one sentence on the reusable pattern, and a rough trigger description. Discard anything that's really just this-project detail wearing a skill's clothes.

**3. Propose via AskUserQuestion.** Multi-select — the user may want more than one, or none of the above (they can always type something else via "Other").

**4. Build each selected skill.** Follow the `my-claude-skills` repo convention exactly (see its top-level `README.md` "스킬 구조" section): `SKILL.md` (required), `template.md` (only if output shape must be fixed), `README.md` (required, human-facing).

**5. Register it.**
- Write the canonical folder into the `my-claude-skills` repo (`/Users/kimjaehyeon/Desktop/1_개발프로젝트/skills/<skill-name>/`)
- Symlink it into `~/.claude/skills/<skill-name>` so it's actually invocable — `ln -s <repo-path>/<skill-name> ~/.claude/skills/<skill-name>`. If a same-named entry already exists there, stop and tell the user instead of overwriting it.
- Add a row for it to the repo's top-level `README.md` skill table

**6. Commit, don't push.** Branch off `main` first (never commit on `main`/`master` directly), commit there, merge into `main` locally if that's this repo's pattern — then **stop and ask before pushing**. Committing/pushing only happens when the user asks for it.

**7. Say what's untested.** New skills built this way are a first draft, not verified against RED-GREEN-REFACTOR pressure scenarios. Say so plainly rather than implying they've been tested.

## Common Mistakes

- Proposing something so tied to this one project's specifics that it can't actually generalize
- Skipping the AskUserQuestion step and just picking a candidate yourself
- Pushing to the remote without being asked, or committing straight on `main`
- Overwriting an existing `~/.claude/skills/<name>` instead of flagging the collision
- Forgetting to add the new skill to the repo's top-level README table
