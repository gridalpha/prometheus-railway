# Blackbox exporter for Railway.
#
# Only the module set differs from upstream, so the published image is used as-is.
# The stock config forces preferred_ip_protocol: ip4 on every HTTP module, which
# cannot probe a *.railway.internal target: private DNS is IPv6-first and the
# container's own 10.x address is not routable from a peer.
FROM prom/blackbox-exporter:latest

COPY blackbox/blackbox.yml /etc/blackbox_exporter/config.yml

EXPOSE 9115

CMD ["--config.file=/etc/blackbox_exporter/config.yml", "--web.listen-address=[::]:9115"]
