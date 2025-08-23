# Minimal base with apt-get
FROM debian:stable-slim

# Install unbound + dnsutils (dig) and clean up
RUN apt-get update \
 && apt-get install -y --no-install-recommends unbound dnsutils ca-certificates \
 && rm -rf /var/lib/apt/lists/*

# Provide a default config so the container can run out-of-the-box.
# You can still override it at runtime by mounting your own unbound.conf.
RUN mkdir -p /etc/unbound && \
    printf '%s\n' \
      'server:' \
      '  interface: 0.0.0.0@5335' \
      '  interface: ::0@5335' \
      '  do-ip4: yes' \
      '  do-ip6: no' \
      '  do-udp: yes' \
      '  do-tcp: yes' \
      '  hide-identity: yes' \
      '  hide-version: yes' \
      '  qname-minimisation: yes' \
      '  prefetch: yes' \
    > /etc/unbound/unbound.conf

EXPOSE 5335/udp 5335/tcp
HEALTHCHECK CMD dig @127.0.0.1 -p 5335 version.bind TXT CHAOS +short || exit 1
ENTRYPOINT ["/usr/sbin/unbound","-d","-c","/etc/unbound/unbound.conf"]
