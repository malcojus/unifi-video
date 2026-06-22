# TODO

## Find and add the source Dockerfile for the deployed image

This repo is currently **deployment-only**: `docker-compose.yaml` pulls the
prebuilt image `malcojus/unifi-video-controller-minimal:latest` from Docker Hub.
There is no build source committed here yet.

The image was almost certainly built from a local clone of the original repo on
another machine, then pushed to Docker Hub manually — so the Dockerfile that
produced it isn't tracked anywhere public.

**Goal:** locate that Dockerfile (and its `run.sh` / `unifi-video.patch`) and add
it here so `docker build .` reproduces the deployed image, making the repo
self-contained.

### What I know so far

- **Closest known source:** <https://github.com/malcojus/UniFi-Video-Controller>
  (a 2023 fork of <https://github.com/pducharme/UniFi-Video-Controller>). It
  contains `Dockerfile`, `run.sh`, `unifi-video.patch`, and an old
  `docker-compose.yaml`.
- **Caveat — it likely does NOT match the deployed image as-is:**
  - That fork's Dockerfile builds `FROM phusion/baseimage:0.11` (the full,
    non-"minimal" pducharme lineage) and was last committed **March 2023**.
  - The Docker Hub image this repo pulls, `...-minimal:latest`, was rebuilt
    **January 2026** — a different, slimmer image with no published build source.
  - So the fork's Dockerfile is a starting point, not a guaranteed match. Verify
    the base image and contents before trusting it.

### When picking this up

1. Find the local clone / Dockerfile actually used to build
   `malcojus/unifi-video-controller-minimal:latest`.
2. Add `Dockerfile` (+ `run.sh`, `unifi-video.patch`, any build assets) to this repo.
3. Confirm `docker build .` produces an image equivalent to the published one.
4. Update `docker-compose.yaml` to offer `build: .` (or document both pull and
   build paths), and update `README.md` (the repo-contents table and Quick start).
