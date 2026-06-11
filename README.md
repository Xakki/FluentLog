# fluent-log

Drop-in [fluent-bit](https://fluentbit.io/) configuration that ships Docker container
logs and on-disk log files to [Graylog](https://www.graylog.org/) over GELF, and exposes
fluent-bit's own metrics to Prometheus.

It normalizes heterogeneous sources (PHP/Monolog JSON, nginx, MariaDB, Redis/KeyDB, raw
stderr) into a consistent GELF record: a Monolog-style level, a correct event timestamp,
and GELF-safe flat fields.

## Install

```sh
composer require xakki/fluent-log
```

Companion PHP loggers that emit the structured logs this config expects:

- Laravel / Monolog — [Xakki/LaraLog](https://github.com/Xakki/LaraLog)
- Plain PHP (PSR only) — [Xakki/PHPErrorCatcher](https://github.com/Xakki/PHPErrorCatcher)

## Pipeline

```
Docker fluentd-driver ─┐
  tag service.<container>│         ┌─ json_default (JSON auto-detect, all gl.*)
                        ├─ rewrite ─┤
tail /var/log/json/*    │  gl.<fmt> ├─ service.d/<fmt>.conf (per-type parsing)
tail mariadb slowlog ───┘          └─ level / timestamp / GELF-flatten
                                         └─ OUTPUT  GELF/HTTP → Graylog
                                            metrics → Prometheus (:2021)
```

1. **Ingest.** Two paths:
   - **Docker logging driver** (`forward` input, `:24224`). Containers with
     `logging.driver: fluentd` send here, tagged `service.<container>`.
   - **File tail** — universal NDJSON (`/var/log/json/*.ndjson`) and the MariaDB
     slowlog (mysqld can't write it to stderr).
2. **Flatten metadata.** Driver labels (`com.docker.compose.*`, `tier`, `log_format`)
   become flat `docker_*` fields. `host`/`hostname`/`docker_profile` are added globally
   from the fluent-bit container's env (`host` is mandatory — otherwise GELF would use the
   docker-bridge IP).
3. **Route by `log_format`** → tag `gl.<log_format>` (see below).
4. **Parse JSON globally.** `json_default` tries every record as JSON and silently leaves
   non-JSON lines intact (marked `log_kind=native`).
5. **Per-type parsing** in `service.d/<log_format>.conf`.
6. **Normalize** level (`level_name`/`level_php`/`syslog_severity`), event timestamp, and
   flatten nested maps/arrays into `parent_child` keys (GELF has no nested fields).
7. **Output** GELF over HTTPS to Graylog; `internal_metrics` to a Prometheus exporter.

## Routing by `log_format`

`log_format` selects the parser set; it is **not** tied to the container or service name.

- Default = the compose service name (`Copy docker_service log_format`, only-if-absent —
  an explicit label overrides it).
- Set it explicitly with a Docker label: `log_format: "<fmt>"`.
- Records are routed to the tag `gl.<log_format>`.

A `log_format` that has no matching `service.d/<fmt>.conf` is rewritten to **`auto`**
(`route_unknown` in `cleanup.lua`) and routed to `gl.auto`, where a generic, JSON-safe
multiline filter joins native stack traces. JSON is already handled by the global
`json_default`, so unmarked services need no configuration. The original service identity
stays in `docker_service`.

> `gl.auto` is a single shared multiline buffer for all unmarked services, so lines from
> different containers can in principle cross-merge. For a noisy multi-line service, set an
> explicit `log_format` label to give it its own tag.

### Supported formats

| `log_format`      | Source                                  | Parsing |
|-------------------|-----------------------------------------|---------|
| `php`             | PHP/Monolog (JSON) + raw stderr         | JSON; multiline join of Fatal/Stack trace/dumps; fpm noise dropped |
| `nginx`           | access JSON + error text                | request split; error-tail fields (client/upstream/…) |
| `mariadb`         | error log (stderr)                      | multiline join; record regex |
| `redis`           | Redis / KeyDB                           | line regex |
| `auto` (fallback) | any unmarked service                    | global JSON + generic multiline |
| `mariadb-slowlog` | slowlog file tail                       | multiline on input |

NDJSON files (`/var/log/json/*.ndjson`) are not a `log_format` of their own — each file's
`log_format` is the filename (or a per-line override), so it routes to a known type or
`auto`. See [JSON file logging](#json-file-logging-ndjson).

## Adding a service

Attach the logging driver (see `docker-fluent.yml` for the `x-logging` anchor and examples)
and label the service:

```yaml
services:
  myapp:
    <<: *_logging
    labels:
      tier: "web"
      log_format: "php"   # omit to use the service name / auto fallback
```

**A brand-new log type** with dedicated parsing needs two edits:

1. Create `fluent-bit/service.d/<fmt>.conf` matching `gl.<fmt>`.
2. Add `<fmt>` to `KNOWN_LOG_FORMAT` in `fluent-bit/cleanup.lua` (otherwise it routes to
   `auto`).

Services that don't need special parsing require neither — `gl.auto` handles JSON and
generic multiline automatically.

## JSON file logging (NDJSON)

Write one JSON object per line to `<JSON_LOG_PATH>/<service>.ndjson`. The filename becomes
`docker_service`/`log_format`; a record may override `log_format` and `tier` per line. Used
for app logs that go to a file instead of stdout/stderr.

## Configuration

Set these in `.env` (see `.env_example`):

| Variable                 | Meaning                                         | Example |
|--------------------------|-------------------------------------------------|---------|
| `TZ`                     | container timezone                              | `UTC` |
| `JSON_LOG_PATH`          | host dir tailed at `/var/log/json`              | `/var/log/` |
| `MYSQL_SLOWLOG_PATH`     | host dir tailed at `/var/log/mysql_logs`        | `/var/log/` |
| `EXT_FLUENT_PORT`        | forward bind (host:port)                        | `127.0.0.1:24224` |
| `EXT_FLUENT_METRIC_PORT` | metrics/health bind                             | `127.0.0.1:2020` |
| `GRAYLOG_HOST`           | Graylog GELF/HTTP host                          | `graylog.example.com` |
| `GRAYLOG_URI`            | GELF endpoint                                   | `/gelf` |
| `GRAYLOG_PORT`           | GELF/HTTP port                                  | `443` |
| `HOST_NAME`              | logical host name (GELF `hostname`)             | `example_host` |
| `HOST_IP`                | host IP (GELF `host` / source)                  | `127.0.0.1` |
| `COMPOSE_PROJECT_NAME`   | project name (container/field prefix)           | `example` |
| `COMPOSE_PROFILES`       | profile → `docker_profile`                      | `dev` |

`HOST_NAME` / `HOST_IP` are typically supplied from a `Makefile`:

```makefile
HOST_NAME ?= $(shell hostname)
# first external IP
HOST_IP   ?= $(shell hostname -I 2>/dev/null | awk '{print $$1}' || echo unknown)
```

If `JSON_LOG_PATH` / `MYSQL_SLOWLOG_PATH` are unset, empty named volumes are mounted so the
stack still starts.

## Log rotation

`logrotate/logrotate.conf` rotates `*.log` (mysql) and `*.ndjson` (json) with
`copytruncate` — mandatory because fluent-bit holds the file inode open on tail; a rename
would detach it. Edit the config, then restart the `logrotate` container (it copies the
config to a root-owned path at startup).

## Ports

| Port   | Purpose |
|--------|---------|
| `24224`| fluentd `forward` input (TCP/UDP) |
| `2020` | HTTP server — health / metrics |
| `2021` | Prometheus exporter (internal; proxied by nginx) |
