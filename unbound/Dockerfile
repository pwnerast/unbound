# Pin here for predictability; move to debian:trixie-slim or stable-slim when ready
ARG BASE=debian:stable-slim
FROM ${BASE}

# Install Unbound + dig + packaged root hints (avoids flaky curl-at-build)
RUN apt-get update \
 && apt-get install -y --no-install-recommends \
      unbound dnsutils dns-root-data ca-certificates \
 && rm -rf /var/lib/apt/lists/*

# Initialize DNSSEC trust anchor (ignore non-fatal exit codes)
RUN mkdir -p /var/lib/unbound \
 && unbound-anchor -a /var/lib/unbound/root.key || true

# Default recursive config. Compose can mount your own to /etc/unbound/unbound.conf
# Note: use the packaged root hints path from dns-root-data
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
  '  auto-trust-anchor-file: "/var/lib/unbound/root.key"' \
  > /etc/unbound/unbound.conf

EXPOSE 5335/udp 5335/tcp

# Simple liveness probe
HEALTHCHECK CMD dig @127.0.0.1 -p 5335 version.bind TXT CHAOS +short || exit 1

ENTRYPOINT ["/usr/sbin/unbound","-d","-c","/etc/unbound/unbound.conf"]
