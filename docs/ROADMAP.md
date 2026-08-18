# Spotify-Box Roadmap

This roadmap describes the planned evolution from a Spotify-only speaker into a small, simple media box that can handle multiple audio sources and, on capable hardware, browser-based video streaming.

The current product principle stays unchanged: the UI must remain simple, touch-friendly, predictable, and provide immediate feedback for every user action.

## Goals

- Keep the current Spotify Connect experience working and reliable.
- Allow audio from other phone apps without requiring a dedicated integration for every music service.
- Add additional first-class audio sources such as AirPlay, internet radio, and local/NAS media.
- Add a dedicated Streaming mode for browser-based services such as YouTube, Netflix, Prime Video, Disney+, and WOW.
- Preserve Raspberry Pi 3 support for the core audio experience.
- Treat browser/video streaming on Raspberry Pi 3 as experimental; target Raspberry Pi 4/5 for the full video experience.
- Keep upgrades safe for devices already installed in the field.

## Non-goals

- Do not reimplement Netflix, Disney+, Prime Video, WOW, or other commercial streaming players.
- Do not attempt to bypass DRM or Widevine restrictions.
- Do not try to turn the Raspberry Pi into a fully compatible certified Chromecast receiver.
- Do not embed a heavy browser engine directly inside the Pygame UI unless the architecture is intentionally revisited later.
- Do not sacrifice the simple kid-friendly UI just to expose every provider feature.

## Target architecture

The application should move from a Spotify-specific playback stack to a source-based media architecture.

```text
Mello UI
   |
   +-- SourceManager
   |      |
   |      +-- SpotifyBackend
   |      +-- BluetoothReceiverBackend
   |      +-- AirPlayBackend
   |      +-- RadioBackend
   |      +-- LocalMediaBackend
   |
   +-- StreamingManager
          |
          +-- YouTube
          +-- Netflix
          +-- Prime Video
          +-- Disney+
          +-- WOW
          +-- future browser providers
```

Audio sources share a common playback model where practical. Browser streaming remains a separate operating mode because the browser needs to own the display and may require different audio/video services.

## Phase 0 - Repository and update ownership

Priority: high

- [ ] Change install/update references that still point to the upstream `emieljanson/mello` repository so new and existing installations use `nicolai-m/Spotify-Box`.
- [ ] Audit `install.sh`, `pi/setup.sh`, `pi/auto-update.sh`, README examples, and migration paths for upstream repository references.
- [ ] Keep migrations idempotent and safe for already deployed devices.
- [ ] Add tests or checks around update-source configuration where practical.

### Exit criteria

A fresh install and an auto-update both stay on this repository without silently switching back to upstream Mello.

## Phase 1 - Playback abstraction

Priority: high

The existing `PlaybackController`, `LibrespotAPIProtocol`, Spotify URI handling, and status models are still Spotify-specific. Before adding more sources, introduce a generic playback boundary.

- [ ] Introduce a generic `PlaybackBackend` protocol/interface.
- [ ] Introduce a provider-neutral `PlaybackStatus` / `NowPlaying` model.
- [ ] Add a `SourceManager` that owns the active source and source switching.
- [ ] Move Spotify-specific URI and repeat-context rules into the Spotify backend.
- [ ] Keep the UI dependent on generic playback state rather than `LibrespotStatus`.
- [ ] Define source capabilities such as `can_seek`, `can_skip`, `has_metadata`, and `has_cover_art`.
- [ ] Add source switching rules that stop/pause the old source before the new source becomes active.
- [ ] Add tests for source switching, stale commands, and playback-state propagation.

### Exit criteria

Spotify behaves exactly as before, but the main UI/controller layer no longer assumes every media item is a Spotify URI.

## Phase 2 - Bluetooth receiver mode

Priority: high

Today Bluetooth is used mainly for output to headphones/speakers. Add the opposite direction so the box can act as a Bluetooth speaker for a phone or tablet.

This immediately enables audio from apps such as YouTube Music, Apple Music, Amazon Music, Deezer, Tidal, SoundCloud, browser audio, podcast apps, and audiobook apps without provider-specific integrations.

- [ ] Add Bluetooth A2DP receiver/sink mode for phones/tablets.
- [ ] Preserve the existing Bluetooth output mode for headphones/speakers.
- [ ] Define how input and output Bluetooth modes coexist and switch safely.
- [ ] Route received Bluetooth audio through PipeWire to the built-in speaker or selected output.
- [ ] Expose connection/source state in the Mello UI.
- [ ] Show AVRCP metadata such as title/artist/album/cover when available.
- [ ] Add pairing/disconnect controls to the existing settings flow.
- [ ] Add required PipeWire/WirePlumber/systemd/sudoers changes through `pi/setup.sh` plus a matching `pi/migrate.sh` migration.

### Exit criteria

A phone can select the box as a Bluetooth speaker and play audio from arbitrary apps while Mello clearly shows that Bluetooth is the active source.

## Phase 3 - Additional audio sources

Priority: medium

### AirPlay audio

- [ ] Integrate an AirPlay audio receiver such as Shairport Sync or an equivalent maintained solution.
- [ ] Route audio through the same PipeWire/output path as the other sources.
- [ ] Forward metadata and cover art to Mello when available.
- [ ] Add service lifecycle and source arbitration to `SourceManager`.

### Internet radio

- [ ] Add configurable radio stations/streams.
- [ ] Support station artwork and basic metadata where available.
- [ ] Keep radio browsing simple and suitable for the existing carousel/launcher design.

### Local and network media

- [ ] Support local audio files from configured folders/USB storage.
- [ ] Evaluate SMB/NFS/DLNA sources for NAS playback.
- [ ] Index metadata and artwork without blocking the UI.
- [ ] Reuse progress memory where it makes sense for long-form audio.

### Exit criteria

Spotify, Bluetooth receiver, AirPlay, radio, and local/network media can coexist behind the same source-selection model.

## Phase 4 - Streaming launcher and browser kiosk mode

Priority: medium

Video streaming should run as a dedicated system mode, not inside Pygame.

Proposed lifecycle:

```text
Mello/Pygame mode
    -> user selects Streaming
    -> pause/stop active audio source
    -> persist Mello state
    -> release display resources
    -> start lightweight compositor/browser in kiosk mode
    -> open selected provider
    -> user exits via protected Home/Back action
    -> stop browser
    -> restore display
    -> restart/resume Mello mode
```

- [ ] Add a clean `Streaming` entry point to the UI without cluttering the normal music experience.
- [ ] Add a provider launcher for YouTube, Netflix, Prime Video, Disney+, WOW, and future providers.
- [ ] Store provider definitions in configuration rather than hard-coding UI behavior.
- [ ] Use isolated persistent browser profiles so logins/cookies survive per provider.
- [ ] Never store service passwords or authentication secrets in the repository.
- [ ] Add a reliable Home/Back escape path that cannot be hidden by a provider website.
- [ ] Handle the physical 720x1280 display rotation so browser content appears as 1280x720 landscape to the user.
- [ ] Add crash/watchdog handling so a broken browser session always returns to Mello.
- [ ] Restore audio routing, touch handling, backlight state, and Mello services after browser exit.
- [ ] Add all browser/compositor/system package changes to `pi/setup.sh` and `pi/migrate.sh`.

### Exit criteria

The user can enter Streaming mode, select a provider, use it fullscreen, and reliably return to the normal Mello UI without rebooting.

## Phase 5 - Provider and DRM validation

Priority: medium

Provider support must be capability-tested on real hardware. A website loading successfully is not enough; protected video playback, audio, touch controls, fullscreen, and resume behavior must all work.

Test matrix:

| Provider | Pi 3 | Pi 4 | Pi 5 | Notes |
|---|---|---|---|---|
| YouTube | experimental | target | target | Validate codec/hardware acceleration |
| Netflix | experimental | target | target | Validate Widevine/DRM and resolution |
| Prime Video | experimental | target | target | Validate Widevine/DRM |
| Disney+ | experimental | target | target | Validate Widevine/DRM |
| WOW | experimental | target | target | Validate browser/DRM compatibility |

- [ ] Build a reusable provider smoke-test checklist.
- [ ] Validate login, profile selection, playback start, pause, seek, audio, fullscreen, and logout behavior.
- [ ] Record actual supported resolution/codec/hardware acceleration per Pi generation.
- [ ] Detect hardware capabilities at runtime and hide/mark unsupported or experimental features honestly.
- [ ] Do not add workarounds that bypass provider DRM or terms of service.

### Hardware policy

- Raspberry Pi 3 remains a supported baseline for the audio-focused product.
- Video/browser streaming on Raspberry Pi 3 is experimental and should be aggressively optimized by stopping unneeded Mello/librespot processes while the browser runs.
- Raspberry Pi 4 or newer is the target for a supported video experience.
- Raspberry Pi 5 is the preferred target for the best responsiveness and future headroom.

## Phase 6 - Streaming UX and parental controls

Priority: medium/low

- [ ] Keep provider selection intentionally small and icon-driven.
- [ ] Add optional parent-controlled enable/disable toggles per streaming provider.
- [ ] Add optional streaming time limits / auto-return behavior.
- [ ] Protect settings and account/logout actions behind the existing hidden/parent interaction pattern.
- [ ] Decide whether browser navigation needs a small persistent system overlay or a gesture/physical button.
- [ ] Make source/provider transitions immediately visible through loading and pressed states.

## Phase 7 - Reliability and production hardening

Priority: ongoing

- [ ] Add health checks for source services and browser mode.
- [ ] Add structured logs around source switches, audio route changes, browser start/exit, DRM failures, and recovery.
- [ ] Add recovery for crashes or power loss during a mode transition.
- [ ] Test nightly auto-update across Pi 3, Pi 4, and Pi 5 where supported.
- [ ] Add migration tests/checklists for any change to apt packages, systemd, sudoers, udev, PipeWire/WirePlumber, browser configuration, or boot/display settings.
- [ ] Measure startup time, memory use, CPU use, dropped frames, and thermal behavior for streaming mode.
- [ ] Preserve the ability to disable experimental features when they prove unreliable on a hardware generation.

## Implementation rules

1. Spotify must not regress while new sources are introduced.
2. New media sources must go through the source/backend abstraction rather than adding provider conditionals throughout `app.py`.
3. Browser streaming is a separate operating mode; do not bolt Chromium into the Pygame render loop.
4. A source switch must be explicit, observable, and reversible.
5. Existing devices must survive updates. Any system-level change needs both fresh-install setup and an idempotent migration.
6. Provider availability is capability-based, not promised by name. DRM/browser compatibility can change outside this project.
7. Keep credentials and cookies out of git. Browser profiles live only on the device.
8. Log failures and gather evidence before adding special-case workarounds.
9. Raspberry Pi 3 performance is a constraint, not an excuse for fragile global optimizations that hurt Pi 4/5.
10. Keep the normal music UI simple even as the underlying architecture becomes more capable.

## Suggested first implementation sequence

1. Fix repository ownership in installer/update paths.
2. Introduce `PlaybackBackend`, provider-neutral playback state, and `SourceManager` while keeping Spotify as the only backend.
3. Add Bluetooth receiver mode.
4. Add AirPlay audio.
5. Add internet radio/local media.
6. Add the Streaming launcher and kiosk-mode lifecycle.
7. Validate YouTube first because it is useful without relying on the full commercial DRM matrix.
8. Validate Netflix, Prime Video, Disney+, and WOW individually on Pi 3/4/5.
9. Add parental controls and hardening after the technical path is proven.

## Documentation ownership

Update this roadmap whenever a phase changes materially, a provider is validated/rejected, or the architecture changes. Completed work should be checked off only after it is merged and verified on representative Raspberry Pi hardware.
