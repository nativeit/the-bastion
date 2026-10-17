# The Bastion — CapRover production image
# Built from the repo root as build context (CapRover uploads the whole project).
# Pinned to a release tag at build via git checkout is NOT done here: the image
# bakes in the working tree CapRover uploads, so deploy from a checked-out tag
# (git checkout v3.24.01) before `caprover deploy`, or set the app's branch.
#
# Differs from docker/Dockerfile.debian12 (which is a BUILD-cache test image that
# runs `install --new-install` at BUILD time): here new-install vs upgrade is
# decided at RUNTIME by the entrypoint against persisted volumes, so host keys and
# accounts survive redeploys.
FROM debian:bookworm-slim

LABEL org.opencontainers.image.title="the-bastion-caprover"
LABEL org.opencontainers.image.description="OVHcloud The Bastion, packaged to run as a CapRover app"

# ---- package install (mirrors the upstream debian12 image, cache-friendly) ----
COPY bin/admin/install-ttyrec.sh \
     bin/admin/install-yubico-piv-checker.sh \
     bin/admin/install-mkhash-helper.sh \
     bin/admin/packages-check.sh \
     /opt/bastion/bin/admin/
COPY lib/shell /opt/bastion/lib/shell/
RUN ["/opt/bastion/bin/admin/packages-check.sh","-i","-s"]
RUN ["/opt/bastion/bin/admin/install-ttyrec.sh","-d"]
RUN ["/opt/bastion/bin/admin/install-yubico-piv-checker.sh","-d"]
RUN ["/opt/bastion/bin/admin/install-mkhash-helper.sh","-d"]

# extra runtime deps NOT pulled by packages-check: cron + a tiny init supervisor.
# (syslog-ng comes from packages-check -s above; openssh-server too.)
RUN apt-get update \
 && DEBIAN_FRONTEND=noninteractive apt-get install -y --no-install-recommends \
        cron ca-certificates \
 && rm -rf /var/lib/apt/lists/*

# locales
RUN echo "en_US.UTF-8 UTF-8" >> /etc/locale.gen && locale-gen

# syslog-ng: read from /dev/log, not /dev/kmsg (same fix as upstream image)
RUN test -e /etc/syslog-ng/syslog-ng.conf && \
    sed -i -re 's=system\(\);=unix-stream("/dev/log");=' /etc/syslog-ng/syslog-ng.conf || true

# ---- bastion code ----
COPY . /opt/bastion

# carry forward the sandbox's UID-collision fix (upstream issue #24)
RUN sed -i -re 's/^"accountUidMax":.+/"accountUidMax": 9999,/;s/^"ttyrecGroupIdOffset":.+/"ttyrecGroupIdOffset": 10000,/' \
        /opt/bastion/etc/bastion/bastion.conf.dist

# sanity check the perl environment
RUN ["/opt/bastion/bin/dev/perl-check.sh"]

# our CapRover entrypoint (supervises sshd + syslog-ng + cron, does runtime
# new-install-or-upgrade against the persisted volumes, wires remote syslog)
COPY deploy/caprover/entrypoint.sh /opt/bastion/deploy/caprover/entrypoint.sh
RUN chmod +x /opt/bastion/deploy/caprover/entrypoint.sh

# SSH listens on 22 INSIDE the container; CapRover publishes host 2222 -> 22
# in host mode (see service-update-override.json) to preserve the client IP.
EXPOSE 22/tcp

HEALTHCHECK --interval=30s --timeout=5s --start-period=30s --retries=3 \
    CMD sshd -t && pgrep -x sshd >/dev/null || exit 1

ENTRYPOINT ["/opt/bastion/deploy/caprover/entrypoint.sh"]
