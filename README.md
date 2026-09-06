# Prometheus on Railway

A production-shaped Prometheus monitoring stack for [Railway](https://railway.com):
Prometheus, Alertmanager and the Blackbox exporter, each built from this one repo.

| Service | Dockerfile | Public | Volume | Listens |
|---|---|---|---|---|
| `prometheus` | `prometheus/Dockerfile` | yes, basic auth | `/prometheus` | `:8080` gateway, `[::]:9090` private |
| `alertmanager` | `alertmanager/Dockerfile` | yes, basic auth | `/alertmanager` | `:8080` gateway, `[::]:9093` private |
| `blackbox-exporter` | `blackbox/Dockerfile` | no | none | `[::]:9115` private |

Each Railway service points at this repo and selects its build with
`RAILWAY_DOCKERFILE_PATH`; every build keeps the repo root as its context.

## Why the images are rebuilt rather than used as-is

* **Neither Prometheus nor Alertmanager has any authentication of its own** beyond a
  bcrypt-hashed basic-auth file, and no Railway variable can compute a bcrypt hash.
  Each image therefore runs a small Caddy gateway on `$PORT` that holds the basic
  auth and proxies to the app on loopback, with the hash derived at boot from
  `AUTH_PASSWORD`.
* **That gateway also supplies the health check.** Prometheus' own probe routes are
  `/-/ready` and `/-/healthy`; Railway's `healthcheckPath` accepts only letters,
  digits, `/` and `_`, so a hyphenated path cannot be configured. The gateway
  exposes an anonymous `/healthz` that rewrites to `/-/ready`, which is a real
  readiness check rather than a static 200.
* **`prometheus.yml`, `alertmanager.yml` and the alerting rules are multi-line
  files.** Railway replaces large variable values with placeholder text, so the
  configs are rendered from small scalar variables by each image's entrypoint.
* **The stock blackbox config pins `preferred_ip_protocol: ip4`**, which cannot probe
  a `*.railway.internal` target — Railway's private network is IPv6-first and the
  container's `10.x` address is not routable from a peer. `blackbox/blackbox.yml`
  ships `*_internal` variants that prefer IPv6, and drops the `icmp` modules, which
  need the `CAP_NET_RAW` Railway containers do not have.

The published `prom/*` images are BusyBox/uclibc rootfs with no package manager, so
their static Go binaries are lifted onto Alpine beside Caddy. Each build asserts
`--version` on every lifted binary, so a non-self-contained binary fails the build
instead of crash-looping a container.

## Environment

### `prometheus`

| Variable | Default | Purpose |
|---|---|---|
| `PORT` | `8080` | Port the Caddy gateway binds. Railway probes it. |
| `AUTH_USERNAME` | `admin` | Basic-auth user for the Prometheus UI and API. |
| `AUTH_PASSWORD` | — | **Required.** The container refuses to start without it rather than publishing an open instance. |
| `EXTERNAL_URL` | unset | `https://${{RAILWAY_PUBLIC_DOMAIN}}`. Used for alert generator links. |
| `ALERTMANAGER_HOST` | `alertmanager.railway.internal:9093` | Where alerts are sent. |
| `BLACKBOX_HOST` | `blackbox-exporter.railway.internal:9115` | Prober used by `PROBE_TARGETS`. |
| `SCRAPE_INTERVAL` | `15s` | Also the rule evaluation interval. |
| `SCRAPE_TIMEOUT` | `10s` | |
| `RETENTION_TIME` | `15d` | |
| `RETENTION_SIZE` | `4GB` | Keep below the Railway volume size; the default suits the 5 GB default volume. |
| `SCRAPE_TARGETS` | unset | Comma-separated `host:port` list scraped as job `railway_services`, e.g. `api.railway.internal:8080`. |
| `METRICS_PATH` | `/metrics` | Path used for `SCRAPE_TARGETS`. |
| `PROBE_TARGETS` | unset | Comma-separated URLs probed through the blackbox exporter as job `blackbox_http`. |
| `PROBE_MODULE` | `http_2xx` | Use `http_2xx_internal` for `*.railway.internal` targets. |
| `DATA_DIR` | `<volume>/data` | TSDB path, deliberately one level below the mount root. |

### `alertmanager`

| Variable | Default | Purpose |
|---|---|---|
| `PORT` | `8080` | Port the Caddy gateway binds. |
| `AUTH_USERNAME` | `admin` | Basic-auth user for the Alertmanager UI. |
| `AUTH_PASSWORD` | — | **Required**, same reason as above. |
| `EXTERNAL_URL` | unset | `https://${{RAILWAY_PUBLIC_DOMAIN}}`. Used in notification links. |
| `GROUP_BY` | `alertname,job` | |
| `GROUP_WAIT` / `GROUP_INTERVAL` / `REPEAT_INTERVAL` | `30s` / `5m` / `4h` | |
| `SLACK_WEBHOOK_URL`, `SLACK_CHANNEL` | unset, `#alerts` | Optional Slack notifications. |
| `WEBHOOK_URL` | unset | Optional generic webhook receiver. |
| `SMTP_SMARTHOST`, `SMTP_FROM`, `SMTP_USERNAME`, `SMTP_PASSWORD`, `SMTP_REQUIRE_TLS`, `ALERT_EMAIL_TO` | unset | Optional email notifications. |

With no notifier configured Alertmanager still receives, groups and displays alerts;
only delivery is absent.

### `blackbox-exporter`

| Variable | Default | Purpose |
|---|---|---|
| `PORT` | `9115` | Railway probes it; the service has no public domain. |

## Local build

```sh
docker build -f prometheus/Dockerfile -t prometheus-railway .
docker build -f alertmanager/Dockerfile -t alertmanager-railway .
docker build -f blackbox/Dockerfile -t blackbox-railway .
```

## Versions

`prom/prometheus:v3` and `prom/alertmanager:v0` are pinned to their major, because
both own an on-disk format that a major bump would make unreadable for anyone
already running the template. The blackbox exporter is stateless and tracks
`latest`.

## Licence

Prometheus, Alertmanager and the Blackbox exporter are Apache-2.0 licensed by the
Prometheus Authors. This repository only adds Railway packaging.
