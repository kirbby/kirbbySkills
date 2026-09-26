# General Deploy Script Pattern

Use this guide when creating a deploy script similar to KirbbyOS `main:scripts/deploy-pi.sh`. The pattern works for T3Code apps and other projects that deploy one locally built artifact or release bundle to a remote Linux host managed by systemd.

## Required Shape

A good deploy script should be a single Bash entry point that:

1. Sets `set -euo pipefail`.
2. Defines overrideable defaults for remote host, remote temp paths, install paths, service user, service name, app root, local artifact path, service file path, target OS/architecture when relevant, and health URL.
3. Creates temporary local files/directories with `mktemp`.
4. Registers a cleanup trap that removes local temp files and closes any SSH master connection.
5. Runs local validation before changing the remote host.
6. Builds the deploy artifact locally.
7. Opens one shared SSH connection using `ControlMaster`, `ControlPath`, and `ControlPersist`.
8. Runs an early `ssh -tt ... "sudo -v"` check so password prompts have a terminal and sudo failures happen before copying artifacts.
9. Copies the artifact and systemd service file to remote temp paths.
10. Generates a remote runner script with a single-quoted heredoc so local variables do not expand too early.
11. Executes the remote runner with `ssh -tt` and required values passed in the remote command environment.
12. Removes remote temp files and prints a clear completion message.

## Local Build Phase

Adapt the validation and build commands to the project:

- Go app: `go test ./...`, then `GOOS=linux GOARCH=arm64 go build -ldflags "$LDFLAGS" -o "$LOCAL_BIN" ./cmd/<app>`.
- Node/T3 app: detect `pnpm-lock.yaml`, `yarn.lock`, or `package-lock.json`; run the matching install command only if the repo expects it, then run the package's build command.
- Rust app: `cargo test`, then build with the target already used by the project, or cross-compile only when the remote platform requires it.
- Python service: run the test suite, package or sync the source according to the repo's existing deployment model, and install dependencies in a virtualenv if that is already how production is run.
- If the app has version metadata, read `VERSION` and `git rev-parse --short HEAD`; append `-dirty` when `git status --porcelain` is non-empty.
- Do not deploy if tests or the build fail.
- Runtime-critical files such as migrations, templates, and default config must not depend on the source checkout path. Embed them in the deploy artifact, or install them under the runtime root with explicit permissions and test from a non-repo working directory.
- On macOS, create tar archives with AppleDouble/xattr metadata disabled, for example `COPYFILE_DISABLE=1 tar --format=ustar ...`, so Linux hosts do not warn about `LIBARCHIVE.xattr.com.apple.provenance`.

Choose the artifact format from the repo:

- If the app builds a standalone server bundle, copy that bundle plus static assets.
- If it deploys source to the remote host, copy a tarball or rsync tree and run production install/build remotely only when that is the established app pattern.
- If Docker is already the app's deployment model, build and ship the image or compose bundle instead of forcing the systemd binary pattern.
- If the project already has release packaging, prefer extending that package instead of inventing a second artifact layout.

## SSH And Copy Phase

Use the KirbbyOS connection reuse pattern:

```bash
SSH_CONTROL_DIR="$(mktemp -d "/tmp/t3code-ssh.XXXXXX")"
SSH_CONTROL_PATH="${SSH_CONTROL_DIR}/c"
SSH_OPTS=(
  -o ControlMaster=auto
  -o ControlPath="${SSH_CONTROL_PATH}"
  -o ControlPersist=10m
)
SSH_MASTER_OPTS=(
  -o ControlMaster=yes
  -o ControlPath="${SSH_CONTROL_PATH}"
  -o ControlPersist=10m
)
```

Then open it once:

```bash
ssh "${SSH_MASTER_OPTS[@]}" -fN "${REMOTE}"
```

Validate sudo over a TTY before copying artifacts:

```bash
ssh "${SSH_OPTS[@]}" -tt "${REMOTE}" "sudo -v"
```

Use `scp "${SSH_OPTS[@]}" ...` or `rsync -e "ssh ..."` for artifact transfer. Keep remote paths under `/tmp` until the remote runner installs them with `sudo install`, `tar`, or `rsync` into the final destination.

## Remote Runner Responsibilities

The remote runner should:

1. Run `sudo -v` before doing work.
2. Create the service user if it does not exist:

```bash
if ! id -u "${SERVICE_USER}" >/dev/null 2>&1; then
  sudo useradd --system --user-group --home "${REMOTE_ROOT}" --shell /usr/sbin/nologin "${SERVICE_USER}"
fi
```

3. Install the artifact.
4. Create persistent directories such as `config`, `data`, `logs`, `run`, and any app-specific workspace/cache directories.
5. Set ownership to the service user and restrict sensitive directories with `chmod 700`.
6. Install the systemd service into `/etc/systemd/system/${SERVICE_NAME}`.
7. Run `sudo systemctl daemon-reload`, `enable`, and `restart`.
8. Run smoke checks.
9. Print `systemctl status` and recent `journalctl` output for debugging.
10. Remove remote temporary files.

## Systemd Service File

Create the service file alongside the deploy script when one is missing. It should normally include:

```ini
[Unit]
Description=<App name>
After=network-online.target
Wants=network-online.target

[Service]
Type=simple
User=<service-user>
Group=<service-user>
EnvironmentFile=-/srv/<app>/config/<app>.env
ExecStart=<runtime command>
Restart=on-failure
RestartSec=5
WorkingDirectory=/srv/<app>

NoNewPrivileges=true
PrivateTmp=true
ProtectSystem=full
ReadWritePaths=/srv/<app>

[Install]
WantedBy=multi-user.target
```

Set `ExecStart` to the app's real runtime command, for example a Go binary daemon command, `node server.js`, or a package-manager-free standalone Next.js server. Keep package managers out of `ExecStart` unless the project already uses that style for production.

## Smoke Checks

Choose checks that prove the deployed project works:

- binary version command when available
- HTTP health endpoint with retries and `--retry-connrefused`
- HTTP status or jobs/API endpoints that exercise app state
- database/log maintenance dry runs when the app provides them
- For SQLite-backed services, do not start a second post-restart CLI smoke check that opens the same database. Prefer the running service's HTTP status endpoint, or run the CLI before restart while the service is stopped.
- SQLite-backed services should configure a busy timeout, and prefer WAL mode when multiple processes may open the DB.
- CLI checks that read restricted runtime config or data should run as the systemd service user, for example `sudo -u "${SERVICE_USER}" env CONFIG=... app status`.
- `systemctl status "${SERVICE_NAME}" --no-pager`
- `journalctl -u "${SERVICE_NAME}" -n 50 --no-pager`

Use `curl --fail --show-error` so broken endpoints fail the deploy script.

## Minimal Skeleton

```bash
#!/usr/bin/env bash
set -euo pipefail

REMOTE_HOST="${REMOTE_HOST:-192.168.1.10}"
REMOTE="${REMOTE_HOST}"
APP_NAME="${APP_NAME:-t3code-app}"
SERVICE_NAME="${SERVICE_NAME:-${APP_NAME}.service}"
SERVICE_USER="${SERVICE_USER:-${APP_NAME}}"
REMOTE_ROOT="${REMOTE_ROOT:-/srv/${APP_NAME}}"
REMOTE_TMP="${REMOTE_TMP:-/tmp/${APP_NAME}}"
REMOTE_SCRIPT="${REMOTE_SCRIPT:-/tmp/${APP_NAME}-deploy.sh}"
LOCAL_ARTIFACT="${LOCAL_ARTIFACT:-./dist/${APP_NAME}}"
LOCAL_SERVICE="${LOCAL_SERVICE:-packaging/systemd/${SERVICE_NAME}}"
REMOTE_SERVICE_TMP="${REMOTE_SERVICE_TMP:-/tmp/${SERVICE_NAME}}"
HEALTH_URL="${HEALTH_URL:-http://127.0.0.1:8080/health}"

LOCAL_REMOTE_SCRIPT="$(mktemp)"
SSH_CONTROL_DIR="$(mktemp -d "/tmp/${APP_NAME}-ssh.XXXXXX")"
SSH_CONTROL_PATH="${SSH_CONTROL_DIR}/c"
SSH_OPTS=(-o ControlMaster=auto -o ControlPath="${SSH_CONTROL_PATH}" -o ControlPersist=10m)
SSH_MASTER_OPTS=(-o ControlMaster=yes -o ControlPath="${SSH_CONTROL_PATH}" -o ControlPersist=10m)

cleanup() {
  ssh "${SSH_OPTS[@]}" -O exit "${REMOTE}" >/dev/null 2>&1 || true
  rm -f "${LOCAL_REMOTE_SCRIPT}"
  rm -rf "${SSH_CONTROL_DIR}"
}
trap cleanup EXIT

echo "==> Running local validation"
<test-command>

echo "==> Building artifact"
<build-command>

echo "==> Opening shared SSH connection to ${REMOTE}"
ssh "${SSH_MASTER_OPTS[@]}" -fN "${REMOTE}"

echo "==> Checking remote sudo access"
ssh "${SSH_OPTS[@]}" -tt "${REMOTE}" "sudo -v"

echo "==> Copying artifact"
scp "${SSH_OPTS[@]}" "${LOCAL_ARTIFACT}" "${REMOTE}:${REMOTE_TMP}"

echo "==> Copying systemd service"
scp "${SSH_OPTS[@]}" "${LOCAL_SERVICE}" "${REMOTE}:${REMOTE_SERVICE_TMP}"

cat >"${LOCAL_REMOTE_SCRIPT}" <<'REMOTE_SCRIPT'
#!/usr/bin/env bash
set -euo pipefail

echo "==> Validating sudo access"
sudo -v

echo "==> Ensuring service user"
if ! id -u "${SERVICE_USER}" >/dev/null 2>&1; then
  sudo useradd --system --user-group --home "${REMOTE_ROOT}" --shell /usr/sbin/nologin "${SERVICE_USER}"
fi

echo "==> Installing artifact"
sudo install -m 0755 "${REMOTE_TMP}" "/usr/local/bin/${APP_NAME}"

echo "==> Preparing runtime directories"
sudo mkdir -p "${REMOTE_ROOT}"/{config,data,logs,run}
sudo chown -R "${SERVICE_USER}:${SERVICE_USER}" "${REMOTE_ROOT}"
sudo chmod 700 "${REMOTE_ROOT}"/{config,data,logs,run}

echo "==> Installing and restarting systemd service"
sudo install -m 0644 "${REMOTE_SERVICE_TMP}" "/etc/systemd/system/${SERVICE_NAME}"
sudo systemctl daemon-reload
sudo systemctl enable "${SERVICE_NAME}"
sudo systemctl restart "${SERVICE_NAME}"

echo "==> Running smoke checks"
curl --fail --show-error --retry 10 --retry-delay 1 --retry-connrefused "${HEALTH_URL}"
systemctl status "${SERVICE_NAME}" --no-pager
journalctl -u "${SERVICE_NAME}" -n 50 --no-pager

sudo rm -f "${REMOTE_TMP}" "${REMOTE_SERVICE_TMP}"
REMOTE_SCRIPT

echo "==> Copying deploy runner"
scp "${SSH_OPTS[@]}" "${LOCAL_REMOTE_SCRIPT}" "${REMOTE}:${REMOTE_SCRIPT}"

echo "==> Installing and smoke testing"
ssh "${SSH_OPTS[@]}" -tt "${REMOTE}" \
  "APP_NAME='${APP_NAME}' REMOTE_TMP='${REMOTE_TMP}' REMOTE_ROOT='${REMOTE_ROOT}' REMOTE_SERVICE_TMP='${REMOTE_SERVICE_TMP}' SERVICE_USER='${SERVICE_USER}' SERVICE_NAME='${SERVICE_NAME}' HEALTH_URL='${HEALTH_URL}' bash '${REMOTE_SCRIPT}'; rm -f '${REMOTE_SCRIPT}'"

echo "==> Deploy complete"
```

Replace placeholders with concrete commands before committing. Do not leave `<test-command>` or `<build-command>` in a finished script.
