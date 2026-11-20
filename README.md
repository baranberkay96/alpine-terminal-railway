# Deploy and Host Alpine Linux 3.24 Web Terminal with Railway

[![Deploy on Railway](https://railway.com/button.svg)](https://railway.com/deploy/alpine-linux-324-web-terminal-browser-sh?referralCode=20H68q)

An Alpine Linux 3.24 shell that opens in any browser. You work in bash as the user `alpine` with sudo, inside tmux, so sessions and long-running jobs survive closed tabs. A generated password protects the login, and your home directory sits on a persistent volume. Git, Python, Node.js 24 and everyday CLI tools come preinstalled in a small musl-based image.

## About Hosting Alpine Linux 3.24 Web Terminal

A bare `alpine` container on Railway has no way in except Railway's own CLI. This community template turns it into a working terminal: ttyd serves the shell over HTTPS, nginx stands in front of it and checks a password that Railway generates at deploy, and tmux keeps your work running between visits. The image builds on the official `alpine` image from Docker Hub, pinned by version and digest. `/home` is a Railway volume, Railway's edge provides HTTPS, and the container restarts automatically if it stops. The password never shows up in logs.

## Common Use Cases

- A small cloud Linux shell for Chromebooks, tablets, locked-down laptops or any machine without SSH
- Testing scripts, builds and packages against Alpine and musl libc before shipping Alpine-based containers
- Running long jobs and scripts in tmux and checking back on them later
- Learning Linux and the command line on a disposable, lightweight machine
- A low-cost always-on workspace for Git, Python and Node.js projects

## Dependencies for Alpine Linux 3.24 Web Terminal Hosting

- A Railway volume mounted at `/home` (included in the template)
- Nothing else: no database, no API keys, no external accounts

### Deployment Dependencies

- Alpine Linux 3.24 release notes: https://alpinelinux.org/posts/alpine-3.24.0-released/
- Official Alpine image on Docker Hub: https://hub.docker.com/_/alpine
- ttyd (web terminal, MIT): https://github.com/tsl0922/ttyd
- tmux: https://github.com/tmux/tmux/wiki
- Railway build variables: https://docs.railway.com/builds/dockerfiles#using-variables-at-build-time
- Railway Fair Use Policy: https://railway.com/legal/fair-use

### Implementation Details

| Service | Source | Purpose |
|---|---|---|
| Terminal | Dockerfile based on `alpine:3.24.4` | nginx basic-auth proxy on port 8080 (public HTTPS domain, `/healthz`), ttyd on loopback, tmux sessions, user `alpine` with sudo, volume at `/home` |

**Plan:** works on every plan, including Free (0.5 GB RAM per service). Measured on Railway with a 0.5 GB memory limit: about 0.025 GB of RAM idle with two open tmux sessions, and about 0.06 GB while a Python venv and an npm package were being installed. The image is about 0.1 GB compressed. What you run in the shell decides the rest: large builds can need more than 0.5 GB, and on Free the kernel then stops that process, not the terminal.

**First login**

1. Open the service's public URL.
2. Your browser asks for a user name and password. The user name is `alpine` (the `USERNAME` variable). The password is the generated `PASSWORD` value on the service's Variables tab.
3. You land in a tmux session named `main`. Close the tab at any time and reopen the URL to resume. For a second, independent session, open `https://{your-domain}/?arg=work`.
4. `sudo -i` gives you a root shell.

**Tools included**: bash with completion, coreutils, git, git-lfs, curl, wget, vim, nano, tmux, htop, ncdu, tree, jq, ripgrep, fd, rsync, sqlite3, zip/unzip, dig, ping, netcat, openssh-client, Python 3.14 with pip and pipx, and Node.js 24 LTS with npm. `npm install -g` installs into `~/.npm-global`, so it works without sudo and persists. Alpine's Python is marked as externally managed: use `python3 -m venv` or `pipx` for packages. Compilers (`build-base`) are not included, to keep the image small.

**More packages**: set the `ALPINE_EXTRA_PACKAGES` variable to a space-separated list of Alpine packages, for example `build-base go postgresql-client`. Railway passes the variable to the Docker build, so the next deploy builds an image with those packages and they survive every redeploy. Names apk cannot install are skipped with a warning in the build and deploy logs, so a typo never fails the deploy. Packages you install with `sudo apk add` in the shell work right away but reset on the next redeploy.

**What persists**: everything under `/home/alpine`: files, Git repositories, shell history, dotfiles, Python virtualenvs and pipx apps in `~/.local`, global npm packages in `~/.npm-global`. Running processes and packages installed with apk reset on every redeploy or restart. tmux keeps sessions alive across browser reconnects, not across container restarts.

**Security**: the shell is never served without a password. With an empty, short (under 12 characters) or overlong `PASSWORD` the service still starts, but the login is locked with a random password nobody knows and the deploy log says how to fix it. Invalid `PORT`, `USERNAME` or `TERMINAL_FONT_SIZE` values fall back to the defaults with a warning in the log. nginx checks the password, which the container keeps only as a SHA-512 crypt hash, and rate-limits attempts. It also rejects WebSocket connections from other sites. ttyd listens on 127.0.0.1 only. The password does not appear in logs, process arguments or the shell environment. To change it, edit `PASSWORD` and redeploy. Browsers cache basic-auth logins until you close them, so use a private window on shared computers.

**Limits**: this is a container, not a VM. There is no OpenRC or systemd, no Docker inside the shell, no raw sockets (`ping` works with `sudo`), and only HTTP(S) on the public domain. Software built for glibc may not run on musl; use Alpine packages or musl builds. Railway's [Fair Use Policy](https://railway.com/legal/fair-use) applies to everything that runs in this shell.

**Scaling**: this is a single-user machine on one volume, so keep one replica. Railway scales RAM and CPU vertically up to your plan limit. For a team, deploy one terminal per person.

**Pinning and upgrades**: the Alpine base image is pinned by version and digest on the `FROM` line of `services/terminal/Dockerfile`. Every fresh build runs `apk upgrade`, so security fixes released after the base image are included. To move to a newer Alpine release, change the tag and digest and redeploy.

### Why Deploy Alpine Linux 3.24 Web Terminal on Railway?

Railway gives the terminal an HTTPS URL, a persistent volume and automatic restarts, with no server to administer. You get a protected Alpine Linux shell from one click and pay only for the little CPU and memory it uses. This is a community template, not affiliated with the Alpine Linux project, the ttyd author or Railway.
