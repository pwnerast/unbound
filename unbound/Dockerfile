# Pin for predictability; switch to debian:trixie-slim or debian:stable-slim when you're ready
ARG BASE=debian:stable-slim
FROM ${BASE}

# Install Unbound, dig, packaged root hints, and unbound-anchor (for users who prefer auto-trust anchors)
RUN apt-get update \
 && apt-get install -y --no-install-recommends \
      unbound unbound-anchor dnsutils dns-root-data ca-certificates \
 && rm -rf /var/lib/apt/lists/*

# Default recursive config that works out-of-the-box in CI:
#  - Uses Debian's packaged root hints and root key (no network fetch on build)
#  - If you mount your own config in compose, this file is ignored.
RUN printf '%s\n' \
  'server:' \
  '  interface: 0.0.0.0@5335' \
  '  interface: ::0@5335' \
  '  do-ip4: yes' \
  '  do-ip6: no' \
  '  do-udp: yes' \
  '  do-tcp: yes' \
  '  verbosity: 1' \
  '  prefetch: yes' \
  '  qname-minimisation: yes' \
  '  hide-identity: yes' \
  '  hide-version: yes' \
  '  root-hints: "/usr/share/dns/root.hints"' \
  '  # Use the packaged trust anchor so the container starts even with no state' \
  '  trust-anchor-file: "/usr/share/dns/root.key"' \
  > /etc/unbound/unbound.conf

EXPOSE 5335/udp 5335/tcp

# Simple liveness probe
HEALTHCHECK CMD dig @127.0.0.1 -p 5335 version.bind TXT CHAOS +short || exit 1

# Straightforward entrypoint (no pre-steps needed for the default config)
ENTRYPOINT ["/usr/sbin/unbound","-d","-c","/etc/unbound/unbound.conf"]
