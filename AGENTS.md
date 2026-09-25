# kirbbySkills agent instructions

This repository collects reusable agent skills that can be shared across
projects and agent runtimes.

- Keep every skill self-contained and portable. Do not add KirbbyOS-specific
  paths, API hosts, credentials, or personal data to a shared skill.
- Preserve upstream source, licence, attribution, and notices when importing a
  third-party skill. Document the original URL and revision in `UPSTREAM.md`.
- Put Codex-native skills under `.codex/skills/<name>/`.
- Put runtime-neutral skills under `skills/<name>/`.
- Always commit and push completed changes to the configured remote before
  reporting the task as complete.
