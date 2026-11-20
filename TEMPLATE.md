# Alpine Linux 3.24 Web Terminal (Browser Shell): Railway template spec

## Listing

| Field | Value |
|---|---|
| Name | **Alpine Linux 3.24 Web Terminal (Browser Shell)** ("3.24" is the pinned stable release) |
| Description | Persistent Alpine Linux 3.24 shell in your browser, password-protected. (71 chars) |
| Category | Other |
| Tags | alpine, terminal, web-terminal, linux, shell (the first five are sent), ttyd, tmux, alpine-3.24, musl |
| Icon | `https://cdn.jsdelivr.net/gh/homarr-labs/dashboard-icons/svg/alpine-linux.svg` (200, image/svg+xml) |

Why: the Alpine niche had about 119 deploys in 30 days (marketplace data, 2026-11). The two "Alpine Linux" market templates run bare `alpine:latest` with `tail -f /dev/null`: no web access, no password, nothing installed. This package brings our Ubuntu/Debian/Kali web terminal design (116, 60, 109) to Alpine.

## Base

Derived from `116-ubuntu-terminal` (same nginx auth proxy, `/healthz`, ttyd auth-header mode, tmux sessions, fail-closed password handling per STANDARD 8.29, build-time extra packages like 109's `KALI_EXTRA_PACKAGES`). See `60-debian-terminal/template/TEMPLATE.md` for the full rationale of the nginx + ttyd setup.

Alpine-specific changes:
- bash is installed first and is the login shell; the user `alpine` (uid 1000) is created with shadow's `useradd`. Alpine has no `/etc/skel` dotfiles, so the Dockerfile writes a `.bash_profile` (loads `.bashrc`) and a `.bashrc` (loads the template defaults).
- ttyd 1.7.7, tini 0.19.0 and Node.js 24.22.3 LTS come from Alpine packages (musl builds; nodejs.org only ships glibc tarballs).
- nginx runs as `nginx`; the htpasswd file is `root:nginx 0640`. Temporary paths are set in the rendered config (`/run/terminal-nginx`), and nginx starts with `-e stderr`, so compiled-in paths do not matter. The same change is used in all four round-12 distro terminals.
- `build-base` is left out to keep the image small (about 0.1 GB compressed); `ALPINE_EXTRA_PACKAGES=build-base` adds it.
- The git prompt script is found by path search (`/usr/share/git-core/git-prompt.sh` on Alpine).
- New homes get mode 0750 (`cp -a /etc/skel/.` had copied 0755 onto the home).

## Pinned versions (checked 2026-11-20)

| Component | Source | Version |
|---|---|---|
| Base image | `alpine:3.24.4@sha256:f59abe656b692d901e184be3445e084786602108599d2a51d229c0f5a48c71bf` | Alpine Linux 3.24.4 |
| ttyd | Alpine community | 1.7.7-r0 |
| Node.js / npm | Alpine main / community | 24.22.3-r0 / 11.12.1-r0 |
| nginx | Alpine main | 1.30.7-r0 |
| tmux | Alpine main | 3.7c-r0 |
| Python | Alpine main | 3.14.10-r1 |
| bash | Alpine main | 5.3.9-r1 |

Every build runs `apk upgrade`, so point releases within 3.24 are picked up.

## Services

### Terminal

| Setting | Value |
|---|---|
| Source | repo `baranberkay96/alpine-terminal-railway`, root `/services/terminal` (Dockerfile) |
| Healthcheck | `/healthz` on `PORT=8080`, timeout 120 s |
| Public domain | yes, target port 8080 |
| Volume | `/home` (home of `alpine` is `/home/alpine`) |
| Restart policy | ON_FAILURE, 10 retries |
| Replicas | 1 |

### Variables

| Variable | Default | Notes |
|---|---|---|
| `PASSWORD` | `${{secret(32)}}` | 12-256 bytes, one line. Otherwise the login is locked with a random password; the deploy still succeeds and the log says how to fix it. |
| `USERNAME` | `alpine` | Invalid names fall back to `alpine` with a warning. |
| `PORT` | `8080` | Invalid values or 7681 fall back to 8080 with a warning. |
| `TERMINAL_TMUX` | `true` | `false`: plain login shell per tab. |
| `TERMINAL_FONT_SIZE` | `15` | 8-40; anything else falls back to 15 with a warning. |
| `ALPINE_EXTRA_PACKAGES` | empty | Build argument. Invalid names and packages apk cannot install are skipped with a warning (build log and deploy log); never fails the build. |
| `TZ` | `UTC` | Shell time zone. |
| `RAILWAY_DEPLOYMENT_DRAINING_SECONDS` | `15` | Grace period for the clean tmux shutdown. |

No required inputs.

## Resource estimate

Railway prices: RAM $10/GB-month, vCPU $20/vCPU-month, volume $0.15/GB-month.

| Service | Idle (measured) | Typical | Notes |
|---|---|---|---|
| Terminal | 0.022-0.026 GB RAM, ~0.005 vCPU (measured, 0.5 GB limit) | 0.06 GB during pip/npm installs (measured), 0.2 GB avg | Peaks come from what users run |
| Volume `/home` | 0.1 GB | 1-3 GB | Projects, venvs, node_modules |

Estimated monthly cost: idle about $0.40, typical about $2.50. Minimum plan: Free (0.5 GB RAM per service). Image: 3.9 MB base plus about 99 MB of apk packages (204 packages, about 275 MB installed), far below the 4 GB Trial/Free image cap.

## Improvements over market templates

1. A browser terminal at all: the market templates only keep a bare container alive for `railway ssh`.
2. Generated `secret(32)` password, nginx basic auth with a SHA-512 crypt hash, rate limit, cross-site WebSocket check, loopback-only ttyd. Fails closed without failing the deploy.
3. The whole `/home` persists (dotfiles, history, venvs, pipx, `npm -g`). Market: `/data` only, `/root` lost.
4. tmux sessions survive closed tabs; `?arg=NAME` opens separate sessions.
5. A non-root user with sudo, bash, and the usual CLI tools instead of BusyBox only.
6. Pinned base image (`3.24.4` + digest) instead of `alpine:latest`; `apk upgrade` at build.
7. `/healthz` healthcheck with `PORT` set; build-time extra packages that never fail the build.

## Verification checklist (done on the pre-publish test deploy, 2026-11-20)

- [x] Build completes; deploy SUCCESS; `/healthz` 200.
- [x] `/` 401 without a password, 401 with a wrong one, 200 with `PASSWORD`; `/token` returns `{"token": ""}`.
- [x] WebSocket `/ws` (subprotocol `tty`) opens with the password (101); 403 with a foreign Origin; 401 with a wrong password.
- [x] In the shell: `whoami` = `alpine`, `sudo -n true` works, `node -v` v24.22.3, npm 11.12.1, Python 3.14.10, git 2.54.1, `sudo ping` works (plain `ping` fails as documented), `PASSWORD` not in the environment.
- [x] `python3 -m venv` + `pip install requests`, `npm install -g cowsay` in a second session (`?arg=work`); the `main` session kept running after its tab closed.
- [x] Files, venv and `~/.npm-global` survived three redeploys.
- [x] `PASSWORD=abc`: deploy SUCCESS, lock warning in the log, every password gets 401. `USERNAME="bad user!"` and `TERMINAL_FONT_SIZE=999` fall back with warnings.
- [x] With a 0.5 GB memory limit (`serviceInstanceLimitsUpdate`), `memory.max` = 500 MB, 0.022-0.026 GB idle.

## Open questions

- Software built for glibc does not run on musl; the README says so.
