# Alpine Linux 3.24 Web Terminal for Railway.
#
# Alpine Linux 3.24 with ttyd serving tmux sessions, behind an nginx
# basic-auth proxy. The shell is bash and runs as the non-root user "alpine"
# (passwordless sudo), and /home is a Railway volume. The base image is pinned
# by version tag and digest: change both and redeploy to upgrade.
FROM alpine:3.24.4@sha256:f59abe656b692d901e184be3445e084786602108599d2a51d229c0f5a48c71bf

# Extra Alpine packages to bake into the image, space separated, for example
# "build-base go postgresql-client". Railway passes the service variable of
# the same name as a build argument, so changing it rebuilds the image with
# those packages. They survive redeploys, unlike `apk add` in the shell.
ARG ALPINE_EXTRA_PACKAGES=""

ENV LANG=C.UTF-8 \
    TZ=UTC

# bash first, so the remaining steps can use it.
RUN apk upgrade --no-cache \
 && apk add --no-cache bash

SHELL ["/bin/bash", "-o", "pipefail", "-c"]

# System packages: the auth proxy (nginx), the web terminal (ttyd 1.7.7 from
# Alpine community), the init (tini), tmux, sudo and the tools people expect
# in a shell. Node.js 24 LTS and npm come from Alpine (musl builds; the
# nodejs.org tarballs are glibc only). build-base (gcc, make) is left out to
# keep the image small: add it with ALPINE_EXTRA_PACKAGES.
RUN apk add --no-cache \
      bash-completion bind-tools ca-certificates coreutils curl fastfetch fd file \
      findutils git git-lfs git-prompt grep htop iproute2 iputils-ping jq less \
      libcap-utils lsof nano ncdu ncurses-terminfo netcat-openbsd nginx nodejs npm \
      openssh-client-default openssl pipx procps-ng py3-pip python3 ripgrep rsync \
      sed setpriv shadow sqlite sudo tar tini tmux tree ttyd tzdata unzip vim \
      wget xz zip \
 && ttyd --version \
 && node --version \
 && npm --version

# Optional extras from ALPINE_EXTRA_PACKAGES, checked as package names first
# so the variable cannot inject shell or apk options. A value typed into the
# deploy form must not fail the build: invalid names and packages that apk
# cannot install are skipped with a warning (here and in the deploy log, from
# /etc/alpine-terminal/skipped-packages), and the rest is installed.
RUN set -eux; \
    install -d -m 0755 /etc/alpine-terminal; \
    : > /etc/alpine-terminal/skipped-packages; \
    read -r -a requested <<< "${ALPINE_EXTRA_PACKAGES//,/ }"; \
    extra=(); \
    for pkg in "${requested[@]}"; do \
      if [[ "$pkg" =~ ^[a-z0-9][a-z0-9+._-]*$ ]]; then \
        extra+=("$pkg"); \
      else \
        echo "WARNING: ALPINE_EXTRA_PACKAGES: skipping invalid package name '${pkg:0:64}'" >&2; \
        printf '%s\n' "${pkg:0:64}" >> /etc/alpine-terminal/skipped-packages; \
      fi; \
    done; \
    if (( ${#extra[@]} > 0 )); then \
      if ! apk add --no-cache "${extra[@]}"; then \
        echo "WARNING: ALPINE_EXTRA_PACKAGES: installing '${extra[*]}' together failed. Trying one package at a time." >&2; \
        for pkg in "${extra[@]}"; do \
          if ! apk add --no-cache "$pkg"; then \
            echo "WARNING: ALPINE_EXTRA_PACKAGES: skipping '${pkg}', apk could not install it." >&2; \
            printf '%s\n' "$pkg" >> /etc/alpine-terminal/skipped-packages; \
          fi; \
        done; \
      fi; \
    fi; \
    chmod 0644 /etc/alpine-terminal/skipped-packages

# Railway's container capability set lacks NET_RAW, and the kernel refuses to
# exec a binary whose file capabilities exceed that set: a ping with
# cap_net_raw fails with "Operation not permitted" for every user, root
# included. Without file capabilities ping falls back to ICMP datagram
# sockets, which work under sudo.
RUN set -eux; \
    caps="$(getcap -r /usr /bin /sbin 2> /dev/null || true)"; \
    for bin in $(printf '%s\n' "$caps" | awk 'NF {print $1}'); do setcap -r "$bin"; done; \
    test -z "$(getcap -r /usr /bin /sbin 2> /dev/null || true)"

# The login user "alpine" (uid 1000, bash, passwordless sudo). Its home is
# created on the volume at first boot from /etc/skel: a .bash_profile that
# loads .bashrc, and a .bashrc that loads this template's shell defaults.
RUN set -eux; \
    install -d -m 0755 /etc/skel; \
    printf '# Login shells (tmux starts one per window) load ~/.bashrc.\n[ -f "$HOME/.bashrc" ] && . "$HOME/.bashrc"\n' \
      > /etc/skel/.bash_profile; \
    printf '# ~/.bashrc: your own settings go below the next line. This file is on the volume.\n\n# Alpine Web Terminal defaults (prompt, history, PATH). Delete these lines to opt out.\nif [ -r /etc/alpine-terminal/bashrc ]; then . /etc/alpine-terminal/bashrc; fi\n' \
      > /etc/skel/.bashrc; \
    useradd --uid 1000 --user-group --shell /bin/bash --no-create-home --home-dir /home/alpine alpine; \
    rm -rf /home/alpine; \
    printf 'alpine ALL=(ALL:ALL) NOPASSWD: ALL\n' > /etc/sudoers.d/90-alpine-terminal; \
    chmod 0440 /etc/sudoers.d/90-alpine-terminal; \
    visudo -cf /etc/sudoers.d/90-alpine-terminal; \
    rm -f /etc/nginx/http.d/default.conf

COPY entrypoint.sh /usr/local/bin/alpine-terminal-entrypoint
COPY terminal-session.sh /usr/local/bin/terminal-session
COPY bashrc.sh /etc/alpine-terminal/bashrc
COPY motd.txt /etc/alpine-terminal/motd
COPY nginx.conf.template /etc/alpine-terminal/nginx.conf.template
COPY tmux.conf /etc/tmux.conf
RUN chmod 0755 /usr/local/bin/alpine-terminal-entrypoint /usr/local/bin/terminal-session \
 && chmod 0644 /etc/alpine-terminal/bashrc /etc/alpine-terminal/motd \
      /etc/alpine-terminal/nginx.conf.template /etc/tmux.conf

EXPOSE 8080

# On Railway, podman-init is PID 1 and this runs under it. tini -s registers as
# a child subreaper, so orphaned processes (the tmux server, background jobs)
# are reaped whether or not tini is PID 1.
ENTRYPOINT ["/sbin/tini", "-s", "--", "/usr/local/bin/alpine-terminal-entrypoint"]
