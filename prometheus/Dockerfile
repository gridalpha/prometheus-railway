# Prometheus for Railway.
#
# The published prom/prometheus image is a BusyBox/uclibc rootfs with no package
# manager, so the authenticating gateway and the config renderer are assembled here
# instead: Prometheus' own static binaries are lifted onto Alpine beside Caddy.
FROM prom/prometheus:v3 AS upstream
FROM caddy:2-alpine AS caddy
FROM alpine:3

RUN apk add --no-cache bash ca-certificates su-exec tini \
 && addgroup -g 65532 -S prometheus \
 && adduser -u 65532 -S -G prometheus -H -s /sbin/nologin prometheus \
 && mkdir -p /etc/prometheus/rules /etc/caddy /var/lib/caddy/config /var/lib/caddy/data /prometheus

COPY --from=upstream /bin/prometheus /bin/prometheus
COPY --from=upstream /bin/promtool /bin/promtool
COPY --from=caddy /usr/bin/caddy /usr/bin/caddy

# Fail the build, not a crash loop, if a lifted binary is not self-contained.
RUN /bin/prometheus --version && /bin/promtool --version && /usr/bin/caddy version

COPY prometheus/rules/ /etc/prometheus/rules/
COPY prometheus/entrypoint.sh /usr/local/bin/entrypoint.sh
RUN chmod +x /usr/local/bin/entrypoint.sh && bash -n /usr/local/bin/entrypoint.sh

ENV XDG_CONFIG_HOME=/var/lib/caddy/config \
    XDG_DATA_HOME=/var/lib/caddy/data

EXPOSE 8080 9090

ENTRYPOINT ["/sbin/tini", "-s", "--"]
CMD ["/usr/local/bin/entrypoint.sh"]
