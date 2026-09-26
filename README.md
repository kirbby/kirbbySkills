# kirbbySkills

Reusable agent skills for use across projects and coding-agent runtimes.

## Included skills

- [`babysit-pr`](.codex/skills/babysit-pr): OpenAI Codex PR watcher. It monitors
  review feedback, CI and mergeability, fixes branch-related problems, and
  continues watching until the pull request closes or needs human help.
- [`unslop`](skills/unslop/SKILL.md): Writing rules that remove common AI tells
  and make drafts more specific, direct, and human.
- [`research`](skills/research/SKILL.md): A bounded, source-backed research
  workflow for current facts and technical investigations.
- [`skill-creator`](skills/skill-creator/SKILL.md): Instructions for creating and
  validating portable agent skills.

- [`ui-ux-pro-max`](skills/ui-ux-pro-max/SKILL.md): UI/UX guidance with searchable
  design data, Python search tools, references, and tests.
- [`create-deploy-script`](skills/create-deploy-script/SKILL.md): Deployment
  guidance for local validation, SSH transfer, systemd services, and smoke checks.

## Layout

- `.codex/skills/` contains skills imported in Codex's native layout.
- `skills/` contains runtime-neutral skills.

Imported skills retain their upstream licence and attribution. See
[`UPSTREAM.md`](UPSTREAM.md).
