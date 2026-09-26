---
name: create-deploy-script
description: "Use when creating or reviewing deploy scripts apps or other projects that need the same deployment shape: test and build locally, copy artifacts over SSH, install or restart a systemd service, and run remote smoke checks using the KirbbyOS deploy pattern."
---

# Create Deploy Script

## Overview

Use this skill to build deploy scripts for apps and other projects using the same operational shape as the KirbbyOS main-branch deploy flow: verify locally, build an artifact, transfer over SSH, install through systemd, restart, smoke-test, and clean up.

Before writing a deploy script, read [references/deploy-script.md](references/deploy-script.md).

## Workflow

1. Inspect the project first: language/runtime, package manager, build command, runtime command, artifact location, health endpoints, service user, persistent data paths, and existing systemd or deploy files.
2. If working in the KirbbyOS repo and the user asks for the main-branch deploy pattern, inspect it read-only with `git show main:scripts/deploy-pi.sh` and related files. Do not checkout or switch branches unless the user explicitly asks.
3. Create or update a Bash deploy script with strict mode, environment overrides, local validation, artifact build, SSH connection reuse, remote install script, systemd restart, smoke checks, and cleanup traps.
4. Keep script defaults obvious and overrideable with environment variables. Avoid hidden hostnames, users, paths, ports, or service names when they are project-specific.
5. Add or update the systemd service file when needed so it matches the deploy script's binary or Node runtime path, working directory, service user, environment file, restart policy, and writable paths.
6. Verify with the app's normal test/build command and, where possible, a shell syntax check such as `bash -n scripts/deploy-*.sh`.

## KirbbyOS Pattern Notes

The KirbbyOS main-branch script is the model, not a template to copy blindly. Preserve these ideas:

- local tests run before any remote mutation
- artifact version and commit metadata are embedded when the app supports it
- runtime-critical files such as migrations/templates/default config are embedded or installed under the runtime root; deployed apps must not depend on a source checkout working directory
- tar archives created on macOS disable AppleDouble/xattr metadata, for example with `COPYFILE_DISABLE=1 tar --format=ustar ...`
- SSH uses a temporary ControlMaster connection so password-based deploys prompt minimally
- the deploy runs an early `ssh -tt ... "sudo -v"` preflight before copying artifacts
- the remote install runs as a copied shell script with `sudo -v` up front and is invoked through `ssh -tt`
- system users and runtime directories are created idempotently
- systemd is reloaded, enabled, restarted, then checked
- smoke checks prefer the running service's HTTP health/status endpoints; for SQLite-backed services, do not start a second post-restart CLI smoke check that opens the same database
- SQLite-backed services use a busy timeout and preferably WAL mode when multiple processes may open the DB
- CLI checks that need restricted runtime config run as the systemd service user and happen before restart or while the service is stopped when they touch SQLite
- temporary local and remote files are removed on success and best-effort cleanup runs on failure
