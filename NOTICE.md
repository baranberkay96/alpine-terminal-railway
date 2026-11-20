# NOTICE: Alpine Linux 3.24 Web Terminal template

## Software installed in the image

| Component | Source | License | How it is used |
|---|---|---|---|
| Alpine Linux 3.24 | `alpine` official image, https://hub.docker.com/_/alpine, tag `3.24.4`, index digest `sha256:f59abe65…` (2026-11-08) | Various free licenses (see each package's license in the Alpine package index) | Base image (`FROM`) and apk packages |
| ttyd 1.7.7 | Alpine package `ttyd` (1.7.7-r0, community) | MIT, Copyright (c) 2016-2022 Shuanglei Tao | Web terminal on loopback, installed unmodified |
| nginx 1.30 | Alpine package `nginx` | BSD-2-Clause | Basic-auth reverse proxy. The config was written for this template |
| tmux 3.7 | Alpine package `tmux` | ISC | Persistent sessions |
| tini 0.19.0 | Alpine package `tini` (community) | MIT | Init and child subreaper |
| Node.js 24 LTS | Alpine package `nodejs` (24.22.3) and `npm` | MIT (bundled components under their own licenses) | Installed unmodified from the Alpine archive |
| bash, coreutils, git, Python 3.14, sudo, ripgrep, fd, fastfetch and the other CLI tools | Alpine packages | Each package's own license (GPL, MIT, BSD, Apache and similar) | Installed unmodified from the Alpine archive |

Changes to installed Alpine files: file capabilities are removed from all binaries (Railway's containers cannot exec binaries whose capabilities they lack; this lets `sudo ping` work), the account `alpine` (uid 1000, bash) is added, `/etc/sudoers.d/90-alpine-terminal` grants it passwordless sudo, and `/etc/skel` gets a `.bash_profile` and a `.bashrc` that load this template's shell defaults. nginx's own `nginx.conf` is replaced at every start by the file this template renders. No package files are edited otherwise.

Optional packages named in `ALPINE_EXTRA_PACKAGES` are installed from the Alpine archive at build time under their own licenses.

Product names belong to their owners. This is a community template, not affiliated with the Alpine Linux project, the ttyd author, the OpenJS Foundation or Railway.

## Upstream files and data checked (2026-11-20)

- Docker Hub tags of `library/alpine`: `3.24.4` (index `sha256:f59abe656b692d901e184be3445e084786602108599d2a51d229c0f5a48c71bf`, amd64 manifest `sha256:0f49f331…`, 3.9 MB compressed), the newest 3.24 point release on 2026-11-20.
- Alpine `v3.24` `main` and `community` `APKINDEX` (x86_64): every package name and version above, and the size of the package set.
- Alpine package contents (pkgs.alpinelinux.org): `/sbin/tini`, `/usr/bin/ttyd`, `/bin/setpriv`, `/usr/share/git-core/git-prompt.sh`.
- The ttyd behaviour this template relies on (auth-header mode `-H`, loopback `-i lo`, the `-c` startup log line that prints the credential) was checked for the Debian Web Terminal template of the same author against ttyd tag `1.7.7`; Alpine ships the same release.

## Market templates reviewed

Railway's marketplace "Alpine Linux" templates (`alpine-linux`, `alpine-linux-1`) run the bare `alpine:latest` image with `tail -f /dev/null` and a `/data` volume, reachable only through Railway's own CLI. Only their configuration and listing figures were used, to size the niche. No files or text from other templates were used.

All files in this package were written for this template, adapted from the same author's Ubuntu, Debian and Kali Linux Web Terminal templates: the Dockerfile, `entrypoint.sh`, `terminal-session.sh`, `bashrc.sh`, `tmux.conf`, `nginx.conf.template`, `motd.txt` and the docs.
