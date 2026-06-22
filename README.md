# UniFi Video Controller (Docker)

A Docker Compose deployment for **Ubiquiti UniFi Video** — the discontinued NVR
software that managed UniFi Video cameras (the airCam / UVC line) before it was
replaced by UniFi Protect.

Ubiquiti [officially deprecated UniFi Video](https://community.ui.com/) and no
longer distributes or supports it. This repo exists so that people still running
the original UVC hardware can keep their controller alive in a container on
modern hosts. It runs the last release, **UniFi Video v3.10.13**.

> ⚠️ **Unsupported, end-of-life software.** UniFi Video receives no security
> updates. Do not expose it to the public internet. Reach it over a VPN/LAN only.

## What's in this repo

This is a **deployment configuration**, not application source code:

| File                  | Purpose                                                        |
| --------------------- | -------------------------------------------------------------- |
| `docker-compose.yaml` | The deployment — ports, volumes, capabilities, env.            |
| `.gitignore`          | Keeps runtime data, recordings, and secrets out of git.        |
| `README.md`           | This file.                                                     |

The container image (`malcojus/unifi-video-controller-minimal:latest`) is a
prebuilt image published on Docker Hub; it is pulled, not built from this repo.
It packages UniFi Video v3.10.13 on an Ubuntu 18.04 base with a `/run.sh`
entrypoint and runs as `PUID=99` / `PGID=100`.

## Prerequisites

- Docker Engine + the Docker Compose plugin.
- A host reachable on your LAN/VPN (UniFi Video is **not** safe to expose
  publicly and cannot sit behind a normal HTTP reverse proxy — see below).
- Disk space for recordings. Footage grows quickly; plan for tens to hundreds
  of GB depending on cameras and retention.

## Quick start

```bash
git clone <this-repo> unifi-video-controller
cd unifi-video-controller

# Edit docker-compose.yaml and set TZ to your timezone (default: Etc/UTC)

docker compose up -d
docker compose logs -f      # watch first-time initialization
```

On first start the container creates `./data/` (config, MongoDB, logs, TLS
keystores) and `./videos/` (recordings). Both are bind-mounted from this
directory and are git-ignored.

Then open the web UI:

```
https://<host-ip>:7443
```

It serves a **self-signed certificate**, so your browser will warn you — that's
expected. Complete the setup wizard, then adopt your cameras.

## Configuration

All configuration is in `docker-compose.yaml`.

### Environment variables

| Variable       | Default   | Notes                                                       |
| -------------- | --------- | ----------------------------------------------------------- |
| `TZ`           | `Etc/UTC` | Set to your timezone, e.g. `America/Toronto`.               |
| `DEBUG`        | `0`       | Set to `1` for verbose container logging.                   |
| `CREATE_TMPFS` | `no`      | Leave as `no`; Compose already mounts the tmpfs cache.      |

### Volumes

| Host path   | Container path                  | Contents                                   |
| ----------- | ------------------------------- | ------------------------------------------ |
| `./data`    | `/var/lib/unifi-video`          | Config, MongoDB, logs, **TLS keystores**.  |
| `./videos`  | `/var/lib/unifi-video/videos`   | Recorded footage.                          |

`videos/` is split out from `data/` so you can point it at separate, larger
storage if needed.

### Capabilities

`DAC_READ_SEARCH` is added so the controller (running as PUID/PGID 99/100
inside the container) can traverse and read the bind-mounted data directory
regardless of host file ownership. Without it, startup can fail with permission
errors on the data volume.

### tmpfs cache

`/var/cache/unifi-video` is mounted as tmpfs (RAM) to cut disk wear from
transient cache files. This is paired with `CREATE_TMPFS=no` so the entrypoint
does not try to mount its own tmpfs on top.

## Ports

UniFi Video uses its own TLS and several non-HTTP streaming ports, so it is
**published directly** rather than proxied. Don't try to route it through
Traefik/nginx/etc. — only the HTTP/HTTPS UI ports would work and live streaming
would break.

| Port   | Protocol | Purpose                       |
| ------ | -------- | ----------------------------- |
| `1935` | TCP      | RTMP streaming                |
| `6666` | TCP      | Live FLV                      |
| `7004` | TCP      | Device discovery / control    |
| `7080` | TCP      | HTTP web UI                   |
| `7442` | TCP      | Camera management             |
| `7443` | TCP      | **HTTPS web UI**              |
| `7444` | TCP      | Live stream (HTTPS)           |
| `7445` | TCP      | Live stream (WS)              |
| `7446` | TCP      | Live stream (WSS)             |
| `7447` | TCP      | RTSP/RTMP recording           |

## Backup & restore

Everything that defines your controller lives in `./data/`:

- `db/` — the MongoDB database (cameras, users, recording metadata).
- `keystore`, `cam-keystore`, `ufv-truststore` — TLS material. **Treat as
  secrets.**
- `system.properties` — controller identity (`uuid`) and port config.

To back up, stop the container and archive `./data/` (and `./videos/` if you
want the footage):

```bash
docker compose stop
tar czf unifi-video-data-backup.tgz data/
docker compose start
```

Restore by extracting the archive into place before starting the container.

> Because `./data/` contains private keys and a live database, it is git-ignored
> and must **never** be committed or published.

## Troubleshooting

- **Permission errors on startup** — confirm `DAC_READ_SEARCH` is present under
  `cap_add` and that `./data` is writable by the container's PUID/PGID.
- **Browser TLS warning** — expected; the UI uses a self-signed certificate.
- **Can't reach the UI** — verify port `7443` is published and not firewalled,
  and that you're connecting over `https://`.
- **Verbose logs** — set `DEBUG=1` and run `docker compose logs -f`.

## License & disclaimer

UniFi Video and all related trademarks are property of Ubiquiti Inc. This repo
contains only deployment configuration and documentation; it ships no Ubiquiti
software. The packaged image is provided as-is for users keeping legacy UVC
hardware running. No warranty; use at your own risk.
