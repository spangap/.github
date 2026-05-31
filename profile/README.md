# spangap

## What is this?

**spangap** is a device-application framework for the Espressif ESP32-S3
that ships in two halves: a firmware component and a paired browser package.
An application built on spangap writes only its own domain code — a sensor
loop, a radio stack, a camera — and inherits inter-task communication, a
filesystem, a config tree, networking, a web stack, logging, a CLI, OTA,
WebRTC, and remote access for free.

The name is the dual-side idea: every feature has a device side and a
browser side, designed together and shipped together. Applications consume
one or both halves and layer only domain code on top.

## What this repo owns

This is the "ur-straddle": the **platform meta-repo**. It contains no
firmware or browser code itself. It owns the build-environment image, the
`spangap` host-side shim, the in-container CLI, and the manifest schemas
that the rest of the straddles use:

```
spangap/
├── Dockerfile      build-env image: ESP-IDF 5.5.4 + Node 20 + Python + cairo/Pillow
├── spangap         host-side shim — resolves workspace + project, docker-execs the CLI
├── cli/spangap     in-container CLI (Python) — install, build, flash, monitor
├── install/spangap user-facing installer for the host shim
├── schemas/        straddle.schema.json — the per-straddle manifest format
└── scripts/        flasher daemon, reallyclean.sh, in-container helpers
```

The firmware and browser code that an application actually consumes lives
in the **sibling straddles**:

| Straddle           | What it adds                                                       |
| ------------------ | ------------------------------------------------------------------ |
| `spangap-core`     | base firmware: ITS, fs, storage, log/CLI/cron, pm                  |
| `spangap-net`      | WiFi, TCP/UDP, TLS, NTP, mDNS                                      |
| `spangap-web`      | HTTPS server, auth, WebRTC + the shared browser UI shell           |
| `spangap-lcd`      | on-device LVGL launcher + per-straddle LCD slices                  |
| `acme`             | Let's Encrypt TLS certificates (HTTP-01 + DNS-01)                  |
| `duckdns`          | DuckDNS dynamic-DNS client                                         |
| `upnp`             | UPnP IGD port-mapping                                              |
| `wg`               | WireGuard tunnel (vendors `trombik/esp_wireguard`)                 |
| `ota`              | signed-manifest OTA updates                                        |

A straddle that can build a firmware image (e.g. `hw-tdeck`, one of the first applications
built on spangap) picks the straddles it needs in its `straddle.yaml` and
the build system stages the right component graph into ESP-IDF.

## How it's shipped

- Each firmware straddle is published as an ESP-IDF managed component
  (`spangap/<name>`) on `components.espressif.com`.
- Each browser straddle is exported as TS/Vue source from its `browser/`
  directory; the consuming app's Vite/Quasar pipeline compiles them in.
- The build-env image is published as `ghcr.io/spangap/build-env:<version>`,
  matching the platform-straddle version (locally tagged
  `spangap-build-env:dev`).

## Installing the host shim

The host-side `spangap` command is a thin shim that finds the workspace
(walks up looking for `spangap.workspace.yaml`, then `straddle.yaml`),
makes sure the build-env container is running, and `docker exec`s into
the in-container CLI. Install with:

```sh
./install/spangap
```

After install, run `spangap` from any directory inside a workspace and the
shim does the rest.

## What spangap solves

The platform exists to take recurring embedded-device pain points off the
application's plate. Each block below is a problem you'd otherwise hand-
roll once per device project; spangap takes a single coherent run at it.

### Wiring tasks together takes over the codebase

On an embedded device every concern is its own FreeRTOS task — WiFi, the
web server, TLS, a radio, storage, a sensor loop. Connecting them is
normally bespoke: a queue here, a task notification there, a poll loop
somewhere else, each with its own timeout and its own backpressure rule.
The glue outweighs the feature.

spangap's **ITS** layer is socket-style point-to-point connections between
tasks (TCP between tasks), with one universal blocking primitive
(`itsPoll(timeout)`) per task. A task has exactly one place it waits and
wakes there for everything: incoming connection, packet, stream bytes, or
computed deadline.

### Some tasks aren't allowed to touch the filesystem

The ESP32-S3 has roughly 512 KB of internal DRAM but 8 MB of PSRAM, and
they are not interchangeable. Any SPI-flash operation disables the PSRAM
cache, so a task running on a PSRAM stack that reads a LittleFS file
crashes. DMA, WiFi and lwIP need *internal* DRAM. SD-card DMA needs
internal buffers and breaks under concurrent writers.

spangap routes all I/O through dedicated DRAM-stack worker tasks
(`fs.cpp`). Any task — PSRAM stack included — calls the `fs_*` API
safely, and the workers serialize SDMMC DMA.

### The device and the browser drift out of sync

Settings sprawl into bespoke REST endpoints. Firmware and UI disagree
about shape and defaults. Adding one setting means touching both sides
plus a save path.

spangap's **storage** subsystem is one in-memory cJSON tree synced over a
single packet-mode `storage:1` channel. `s.*` is persisted *and* synced
to the browser; `secrets.*` is persisted but **never** sent to the
browser; everything else is ephemeral. Defaults are self-registering and
version-gated.

### The browser app gets rebuilt from scratch every time

Every connected device needs the same furniture: a settings UI bound to
live config, a log viewer, a CLI console, a live media session, an auth
flow. spangap ships **spangap-browser** with all of that as Vue 3 + Quasar
source: config-bound controls, FloatingWindow, WebRTC session manager,
storage sync, log/CLI rendering, auth flow, menu registry.

### A device behind NAT is unreachable and has no real TLS

spangap's remote-access stack (`acme` + `duckdns` + `upnp`, optionally
`wg`) lets a device behind NAT obtain a real TLS cert and be reachable
from anywhere. WebRTC carries everything else: one `RTCPeerConnection`
and one DTLS handshake per browser tab carry every data path — config,
log, CLI, app media. The WebRTC task is content-free: a DataChannel
labelled `<task>:<n>` routes to ITS port `n` on that task.

### You still have to operate the thing after it ships

A log task hooks ESP-IDF's `vprintf`, tags every line with its
originating task, keeps a DRAM ring buffer, and fans out to serial,
the browser, and an optional log file. A CLI line editor with history
and registrable commands, a minute-resolution cron that survives deep
sleep, and signed-manifest OTA round it out.

## Conventions for a consumer

Most apps consume `spangap-core` plus one or more of the other
straddles. In `app_main()`:

```cpp
#include "log.h"
#include "fs.h"
#include "storage.h"
#include "its.h"
#include "pm.h"
#include "cli.h"
#include "net.h"        // if using spangap-net
#include "web.h"        // if using spangap-web
#include "tls.h"
#include "ntp.h"
#include "ota.h"
#include "ota_pubkey.h" // your app's OTA verification key

extern "C" void app_main() {
    pmInit();
    logInit();
    fs_init();
    storageInit();
    itsInit();
    cliInit();
    netInit();
    webInit();
    tlsInit();
    ntpInit();
    otaInit(OTA_PUBKEY_PEM, OTA_PUBKEY_PEM_LEN);
    // ... your application init here
}
```

Each subsystem self-registers its CLI commands, storage defaults, cron
entries, and WebSocket / HTTP handlers as part of its init.

## Status

Phase 1 of the straddles plan is in place: the old monolithic
`spangap-core` tree has been split into `spangap-core` (base runtime),
`spangap-net` (networking), `spangap-web` (HTTPS/auth/WebRTC + UI shell),
and `spangap-lcd` (on-device launcher), with `acme`, `duckdns`, `upnp`,
`wg`, and `ota` as add-on straddles. The reticulous family of straddles
(in the sibling [reticulous](https://github.com/reticulous) GitHub org)
is the first production consumer.

## Read next

- [INTERNALS.md](INTERNALS.md) — platform-wide architecture, conventions,
  recipes, gotchas, ESP-IDF specifics, partition layout.
- The individual straddle READMEs for each piece you consume.

## Hardware

ESP32-S3 family, dual-core, with **PSRAM** (the platform assumes PSRAM
exists). Flash and PSRAM sizes are not pinned — the partition layout is
the only size-dependent piece and apps may override it. No other
Espressif chip is supported today, though nothing in the architecture is
S3-specific.

## On Claude Code

The overwhelming majority of this project has been written by Claude
Code, over the course of about a month. A project of this scope would
have been an order of magnitude more work for a single person without
LLMs. That said, every `.h` file has been read and iterated on by hand,
and almost every architectural decision is human. The browser code is
mostly Claude's so far. The doc files (this one included) are written
to stay human-readable and to help further development, whether by
humans or AI tools.

### Security note

The design has had separate context spent on security and is at least
*planned* to be securable, but this code as it stands today should not be
used where your life depends on it being unhackable.
