---
name: skill-creator
description: Create, update, validate, or remove reusable agent skills. Use when the user asks to make an agent better at a task, add a reusable workflow, teach an agent a capability, create a skill, change an existing skill, or inspect available project skills.
---

# Skill creator

Use this skill to add or maintain reusable project skills.

## Scope

Create project skills in the host project's skill directory, commonly:

```text
skills/<skill-name>/SKILL.md
```

Use lowercase hyphen-case names. The folder name and frontmatter `name` must match. Follow a host runtime's required directory if it differs.

Do not assume access to host-specific skill tooling unless it is present in the current environment.

## Skill Shape

Every skill needs one `SKILL.md` file:

```md
---
name: example-skill
description: Clear trigger description. Include what the skill does and when to use it.
---

# Example Skill

Concise instructions for doing the task.
```

Keep the frontmatter to `name` and `description` unless there is a concrete reason to add more.

## Workflow

1. Inspect existing skills:

```bash
find skills -maxdepth 2 -name SKILL.md -print
```

2. Choose a short hyphen-case skill name.
3. Create `skills/<skill-name>/SKILL.md`, or the equivalent location required by the host runtime.
4. Put all trigger wording in the `description`; the body loads only after the skill is selected.
5. Keep the body procedural and concise. Include paths, commands, templates, and decision rules that the agent would otherwise rediscover.
6. Avoid broad personality rules, temporary notes, unfinished placeholders, and duplicated content from unrelated skills.
7. Validate the skill structure.
8. Tell the user that existing sessions may need a new conversation, reset, or runtime reload before reliably seeing the new skill.

## Validation

If the host runtime provides a skill validation helper, use it. Otherwise, manually check:

- `SKILL.md` exists.
- Frontmatter starts and ends with `---`.
- `name` is lowercase hyphen-case.
- `description` is present, specific, and under 1024 characters.
- The body has no unfinished placeholders.

## Edit Rules

- Read the existing skill first before changing it.
- Preserve unrelated instructions.
- Prefer updating an existing skill over adding a near-duplicate.
- Delete generated `agents/openai.yaml` files unless the project starts using them consistently.
- Do not create README files, changelogs, or extra documentation unless the user explicitly asks.
- Add scripts or references only when the workflow is too fragile or long to keep in `SKILL.md`.

## Runtime notes

Skill discovery is runtime-specific. Confirm the host runtime's skill directory and refresh behavior before claiming a new skill is available.

After adding or editing a skill, active sessions may keep their previous resource set. Start a new conversation, reset the session, or follow the host runtime's reload procedure when needed.

## Reporting

After creating or editing a skill, report:

- skill name
- skill path
- what behavior it teaches
- whether validation passed
- whether a new session or runtime reload is needed
