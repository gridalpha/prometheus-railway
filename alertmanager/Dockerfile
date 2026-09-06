# Alertmanager for Railway.
#
# Same shape as the Prometheus image: the published prom/alertmanager rootfs is
# BusyBox/uclibc with no package manager, so the static binaries are lifted onto
# Alpine beside Caddy, which supplies the basic auth Alertmanager does not have.
FROM prom/alertmanager:v0 AS upstream
FROM caddy:2-alpine AS caddy
FROM alpine:3

RUN apk add --no-cache bash ca-certificates su-exec tini \
 && addgroup -g 65532 -S alertmanager \
 && adduser -u 65532 -S -G alertmanager -H -s /sbin/nologin alertmanager \
 && mkdir -p /etc/alertmanager /etc/caddy /var/lib/caddy/config /var/lib/caddy/data /alertmanager

COPY --from=upstream /bin/alertmanager /bin/alertmanager
COPY --from=upstream /bin/amtool /bin/amtool
COPY --from=caddy /usr/bin/caddy /usr/bin/caddy

# Fail the build, not a crash loop, if a lifted binary is not self-contained.
RUN /bin/alertmanager --version && /bin/amtool --version && /usr/bin/caddy version

COPY alertmanager/entrypoint.sh /usr/local/bin/entrypoint.sh
RUN chmod +x /usr/local/bin/entrypoint.sh && bash -n /usr/local/bin/entrypoint.sh

ENV XDG_CONFIG_HOME=/var/lib/caddy/config \
    XDG_DATA_HOME=/var/lib/caddy/data

EXPOSE 8080 9093

ENTRYPOINT ["/sbin/tini", "-s", "--"]
CMD ["/usr/local/bin/entrypoint.sh"]
