# Pin for predictability; change when you're ready (trixie-slim or stable-slim are fine too)
ARG BASE=debian:stable-slim
FROM ${BASE}

# Install Unbound + dig + packaged root hints (no flaky network fetch at build)
RUN apt-get update \
 && apt-get install -y --no-install-recommends \
      unbound dnsutils dns-root-data ca-certificates \
 && rm -rf /var/lib/apt/lists/*

# Minimal default recursive config (you can override via bind mount in compose)
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

# Runtime entrypoint: ensure trust anchor exists, then start Unbound
RUN printf '%s\n' \
'#!/bin/sh' \
'set -eu' \
'mkdir -p /var/lib/unbound' \
'# Create DNSSEC root key if missing (ignore non-fatal failures, Unbound will try again later too)' \
'if [ ! -s /var/lib/unbound/root.key ]; then' \
'  unbound-anchor -a /var/lib/unbound/root.key || true' \
'fi' \
'# Quick sanity: show version line in logs' \
'unbound -V || true' \
'exec /usr/sbin/unbound -d -c /etc/unbound/unbound.conf' \
> /usr/local/bin/docker-entrypoint.sh \
 && chmod +x /usr/local/bin/docker-entrypoint.sh

EXPOSE 5335/udp 5335/tcp

# Simple liveness probe
HEALTHCHECK CMD dig @127.0.0.1 -p 5335 version.bind TXT CHAOS +short || exit 1

ENTRYPOINT ["/usr/local/bin/docker-entrypoint.sh"]
