# You can pin the base OS here. Change to debian:trixie-slim or debian:stable-slim if you prefer.
ARG BASE=debian:stable-slim
FROM ${BASE}

# Install Unbound + dig, keep image slim
RUN apt-get update \
 && apt-get install -y --no-install-recommends \
      unbound dnsutils ca-certificates curl \
 && rm -rf /var/lib/apt/lists/*

# Fetch current root hints so the default config can resolve recursively
RUN mkdir -p /etc/unbound /var/lib/unbound \
 && curl -fsSL https://www.internic.net/domain/named.root -o /etc/unbound/root.hints \
 # Initialize DNSSEC trust anchor (ignore non-fatal exit codes)
 && unbound-anchor -a /var/lib/unbound/root.key || true

# Provide a sane default recursive config. Your compose can override it by mounting your own.
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
  '  root-hints: "/etc/unbound/root.hints"' \
  '  auto-trust-anchor-file: "/var/lib/unbound/root.key"' \
  > /etc/unbound/unbound.conf

EXPOSE 5335/udp 5335/tcp

# Simple liveness probe
HEALTHCHECK CMD dig @127.0.0.1 -p 5335 version.bind TXT CHAOS +short || exit 1

ENTRYPOINT ["/usr/sbin/unbound","-d","-c","/etc/unbound/unbound.conf"]
