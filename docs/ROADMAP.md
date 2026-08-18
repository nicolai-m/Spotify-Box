# Spotify-Box Roadmap

This roadmap describes the planned evolution from a Spotify-only speaker into a small, simple media box that can handle multiple audio sources and, on capable hardware, optional browser-based video streaming.

The current product principle stays unchanged: the UI must remain simple, touch-friendly, predictable, and provide immediate feedback for every user action. Video must remain optional so the same software can also be operated permanently as an audio-only streaming box.

## Goals

- Keep the current Spotify Connect experience working and reliable.
- Allow audio from other phone apps without requiring a dedicated integration for every music service.
- Add additional first-class audio sources such as AirPlay, internet radio, and local/NAS media.
- Add an optional dedicated Streaming mode for browser-based services such as YouTube, Netflix, Prime Video, Disney+, and WOW.
- Make video providers configurable so additional browser-based streaming services can be added and managed without changing application code.
- Allow video/Streaming mode to be globally disabled so the device can be locked to audio-only operation.
- Add a local web administration interface reachable through the box IP/hostname for configuration and device management.
- Allow the web admin to enable/disable video, manage video providers, configure supported settings, inspect status, trigger updates, restart, and shut down the box.
- Allow the owner to optionally protect the local web administration interface with a password that can be set, changed, or removed from the admin settings.
- Preserve Raspberry Pi 3 support for the core audio experience.
- Treat browser/video streaming on Raspberry Pi 3 as experimental; target Raspberry Pi 4/5 for the full video experience.
- Keep upgrades safe for devices already installed in the field.

## Non-goals

- Do not reimplement Netflix, Disney+, Prime Video, WOW, or other commercial streaming players.
- Do not attempt to bypass DRM or Widevine restrictions.
- Do not try to turn the Raspberry Pi into a fully compatible certified Chromecast receiver.
- Do not embed a heavy browser engine directly inside the Pygame UI unless the architecture is intentionally revisited later.
- Do not sacrifice the simple kid-friendly UI just to expose every provider feature.
- Do not expose the administration interface directly to the public internet by default.
- Do not implement web-admin actions by accepting arbitrary shell commands from HTTP requests.

## Target architecture

The application should move from a Spotify-specific playback stack to a source-based media architecture with a separate local administration plane.

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
   |      |
   |      +-- VideoPolicy / video_enabled
   |      +-- ProviderRegistry
   |             |
   |             +-- built-in providers
   |             +-- admin-defined providers
   |
   +-- DeviceControl / Settings
          |
          +-- local Web Admin
          +-- optional AdminAuth
          +-- provider management
          +-- update/restart/shutdown
          +-- WiFi/Bluetooth/settings
          +-- health/status
```

Audio sources share a common playback model where practical. Browser streaming remains a separate operating mode because the browser needs to own the display and may require different audio/video services. The local web admin should call typed application/device-control APIs instead of reaching directly into Pygame internals or executing arbitrary commands.

Video providers should be data-driven. The launcher must read provider definitions from a registry/configuration layer instead of hard-coding one button and launch path per service.

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

## Phase 4 - Streaming launcher, provider registry, video policy, and browser kiosk mode

Priority: medium

Video streaming should run as a dedicated system mode, not inside Pygame. It must also be possible to disable it globally and operate the device as an audio-only box.

Proposed lifecycle:

```text
Mello/Pygame mode
    -> check video_enabled policy
    -> user selects Streaming
    -> choose enabled provider from ProviderRegistry
    -> pause/stop active audio source
    -> persist Mello state
    -> release display resources
    -> start lightweight compositor/browser in kiosk mode
    -> open provider start URL with provider-specific browser profile
    -> user exits via protected Home/Back action
    -> stop browser
    -> restore display
    -> restart/resume Mello mode
```

### Video policy

- [ ] Add persistent `video_enabled` / audio-only policy state.
- [ ] When video is disabled, remove/hide the Streaming launcher from the child-facing UI and reject all attempts to start browser streaming.
- [ ] Do not start or keep browser/compositor processes running while video is disabled.
- [ ] Allow the policy to be changed from protected local settings and from the web administration interface.
- [ ] If video is disabled remotely while Streaming mode is active, safely close the browser session and return to Mello/audio mode.
- [ ] Keep per-provider enable/disable settings separate from the global video switch.

### Provider registry

- [ ] Introduce a persistent `ProviderRegistry` or equivalent provider configuration service.
- [ ] Ship sensible built-in provider presets for YouTube, Netflix, Prime Video, Disney+, and WOW without making those names special cases in the launcher.
- [ ] Allow additional browser-based video providers to be added without a code change.
- [ ] Define provider fields such as stable ID, display name, start URL, enabled state, order, optional icon/artwork, browser-profile identifier, and optional capability/compatibility notes.
- [ ] Allow provider definitions to be edited, enabled/disabled, reordered, and removed from the web admin.
- [ ] Distinguish shipped presets from user-added providers so presets can receive safe default updates without overwriting owner customizations.
- [ ] Allow a removed built-in preset to be restored to its default definition.
- [ ] Keep one isolated persistent browser profile per provider so logins/cookies survive and different providers do not unnecessarily share session state.
- [ ] Validate configured launch URLs before saving/launching. Accept only intended web schemes such as `https://` (and `http://` only when explicitly allowed for trusted local services); reject dangerous schemes such as `file:`, `javascript:`, `data:`, or arbitrary command/protocol handlers.
- [ ] Treat custom providers as unverified until tested; adding a URL must never imply that DRM, playback, touch, or hardware acceleration is supported.
- [ ] Never collect or store provider account passwords in Mello settings. Provider login remains inside the provider browser profile.

### Kiosk launcher

- [ ] Add a clean `Streaming` entry point to the UI without cluttering the normal music experience.
- [ ] Render the streaming launcher from enabled provider definitions rather than hard-coded provider buttons.
- [ ] Store provider definitions in device-local configuration and keep a migration/versioning strategy for future schema changes.
- [ ] Use isolated persistent browser profiles so logins/cookies survive per provider.
- [ ] Never store service passwords or authentication secrets in the repository.
- [ ] Add a reliable Home/Back escape path that cannot be hidden by a provider website.
- [ ] Handle the physical 720x1280 display rotation so browser content appears as 1280x720 landscape to the user.
- [ ] Add crash/watchdog handling so a broken browser session always returns to Mello.
- [ ] Restore audio routing, touch handling, backlight state, and Mello services after browser exit.
- [ ] Add all browser/compositor/system package changes to `pi/setup.sh` and `pi/migrate.sh`.

### Exit criteria

When video is enabled, the user can enter Streaming mode and launch any enabled provider from the registry. A parent/admin can add another compatible browser service without changing application code, and it appears in the launcher after being enabled. When video is disabled, the same device behaves as an audio-only box and no browser streaming path can be started.

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
| Custom provider | unverified | unverified | unverified | Validate individually after configuration |

- [ ] Build a reusable provider smoke-test checklist.
- [ ] Validate login, profile selection, playback start, pause, seek, audio, fullscreen, and logout behavior.
- [ ] Record actual supported resolution/codec/hardware acceleration per Pi generation for built-in providers.
- [ ] Allow custom provider entries to store owner-visible compatibility notes without representing them as officially validated.
- [ ] Detect hardware capabilities at runtime and hide/mark unsupported or experimental features honestly.
- [ ] Do not add workarounds that bypass provider DRM or terms of service.

### Hardware policy

- Raspberry Pi 3 remains a supported baseline for the audio-focused product.
- Video/browser streaming on Raspberry Pi 3 is experimental and should be aggressively optimized by stopping unneeded Mello/librespot processes while the browser runs.
- Raspberry Pi 4 or newer is the target for a supported video experience.
- Raspberry Pi 5 is the preferred target for the best responsiveness and future headroom.
- Audio-only mode must remain fully supported on Pi 4/5 as well; video-capable hardware must not force the feature to be enabled.

## Phase 6 - Local web administration

Priority: medium

Add a lightweight local administration interface so a parent/admin can configure the box from a phone, tablet, or computer without navigating the child-facing touchscreen UI.

Expected access:

```text
http://<box-hostname>.local/
```

or via the current LAN IP address.

### Core controls

- [ ] Show device status: hostname, IP address, software version, uptime, active source, playback state, update state, and relevant health information.
- [ ] Add a global Video / Streaming enable-disable switch.
- [ ] Add per-provider enable-disable controls for browser streaming services.
- [ ] Add a Video Providers administration page/list.
- [ ] Allow a provider to be added with at least a name and start URL, plus optional icon/artwork.
- [ ] Allow providers to be edited, enabled/disabled, reordered, and removed.
- [ ] Allow built-in provider presets to be restored to defaults.
- [ ] Show whether a provider is built-in, custom, validated, experimental, unsupported, or not yet tested where that information exists.
- [ ] Apply provider-list changes to the touchscreen launcher without requiring a reboot where practical.
- [ ] Allow audio-only operation to be selected and persisted across reboots/updates.
- [ ] Expose safe audio settings such as speaker/Bluetooth volume levels, auto-pause, and progress-memory settings.
- [ ] Expose source controls where useful: Spotify status, Bluetooth pairing/connection, AirPlay enablement, radio configuration, and local-media settings as those features are implemented.
- [ ] Expose WiFi/network configuration with clear warnings that switching networks can disconnect the current browser session.
- [ ] Add `Check for updates` and `Install update` actions with visible progress/result state.
- [ ] Add restart and shutdown actions with explicit confirmation.
- [ ] Consider factory reset only behind an additional confirmation/protection step.
- [ ] Keep settings synchronized between touchscreen and web UI through one shared settings/service layer.
- [ ] Add an Admin Security section where the owner can optionally set a password, change it, or remove password protection again.
- [ ] Clearly show whether the web admin is currently protected or running without a password.

### Optional password protection

Password protection is optional because some deployments may only use the box inside a trusted home LAN. The owner must be able to decide whether the local web admin requires authentication.

- [ ] Allow initial setup and the Admin Security page to enable password protection without editing files or using SSH.
- [ ] When a password is configured, require authentication before entering the administration interface and before any configuration/system action.
- [ ] Provide an authenticated way to change the current password.
- [ ] Provide an authenticated, explicitly confirmed way to remove the password and return to unprotected LAN-only administration.
- [ ] Store only a modern salted password hash; never persist the plaintext password.
- [ ] Use session-based authentication with logout and reasonable session expiry when password protection is enabled.
- [ ] Rate-limit failed login attempts when password protection is enabled.
- [ ] If no password is configured, show a visible warning in the admin UI that anyone with access to the local network may be able to administer the box.
- [ ] Password-protection state and the hash must survive normal reboots/updates while remaining device-local.

### Security and architecture rules

- [ ] Bind the admin interface to local/LAN access by default; never intentionally expose it to the public internet during installation.
- [ ] Authentication is optional, but when an admin password is configured it must protect the entire admin interface and all privileged actions consistently.
- [ ] Store password material safely as a hash, never plaintext in git or logs.
- [ ] Protect state-changing requests against CSRF and accidental repeated submissions regardless of whether password protection is enabled.
- [ ] Rate-limit authentication attempts and security-sensitive actions where practical.
- [ ] Use explicit typed/allow-listed actions for update, reboot, shutdown, networking, provider management, and service control. HTTP input must never become an arbitrary shell command.
- [ ] Validate provider URLs and provider IDs server-side; never trust browser-side validation alone.
- [ ] Separate read-only status endpoints from privileged mutation endpoints.
- [ ] Never expose Spotify credentials, browser cookies, Widevine/provider sessions, tokens, or other sensitive account information through the admin UI/API.
- [ ] Require explicit confirmation for update, reboot, shutdown, reset, network changes, and destructive provider operations where appropriate.
- [ ] Log what action was requested and whether it succeeded, but never log passwords, cookies, tokens, or provider session data.
- [ ] Make the web-admin service resilient to Mello/Pygame or browser-mode restarts so remote administration remains available when practical.
- [ ] Add any new web service/systemd/firewall/sudoers dependencies to both fresh-install setup and `pi/migrate.sh`.

### Exit criteria

A parent/admin on the same local network can open the box by hostname/IP, inspect its state, toggle audio-only/video mode, manage the provider list, change supported settings, update the software, and restart or shut down the box without SSH. The owner can optionally enable password protection directly in the admin interface, later change or remove it, and the interface clearly reports whether it is protected. Additional compatible video services can be added from the admin interface without a code deployment. The interface is not publicly exposed by default and cannot execute arbitrary commands.

## Phase 7 - Streaming UX and parental controls

Priority: medium/low

- [ ] Keep provider selection intentionally small and icon-driven.
- [ ] Render provider order and visibility from the provider registry.
- [ ] Add optional parent-controlled enable/disable toggles per streaming provider.
- [ ] Treat the global video lock as a parent/admin policy, not a child-facing toggle.
- [ ] Add optional streaming time limits / auto-return behavior.
- [ ] Protect settings and account/logout actions behind the existing hidden/parent interaction pattern.
- [ ] Decide whether browser navigation needs a small persistent system overlay or a gesture/physical button.
- [ ] Make source/provider transitions immediately visible through loading and pressed states.
- [ ] Ensure web-admin changes are reflected on the touchscreen without requiring a reboot where possible.

## Phase 8 - Reliability and production hardening

Priority: ongoing

- [ ] Add health checks for source services, web administration, and browser mode.
- [ ] Add structured logs around source switches, audio route changes, browser start/exit, provider configuration changes, video-policy changes, web-admin system actions, DRM failures, and recovery.
- [ ] Add recovery for crashes or power loss during a mode transition.
- [ ] Test nightly auto-update across Pi 3, Pi 4, and Pi 5 where supported.
- [ ] Add migration tests/checklists for any change to apt packages, systemd, sudoers, udev, PipeWire/WirePlumber, browser configuration, provider schema, web-admin services, or boot/display settings.
- [ ] Measure startup time, memory use, CPU use, dropped frames, and thermal behavior for streaming mode.
- [ ] Preserve the ability to disable experimental features when they prove unreliable on a hardware generation.
- [ ] Verify shutdown/restart/update actions cannot leave the device in an inconsistent state.

## Implementation rules

1. Spotify must not regress while new sources are introduced.
2. New media sources must go through the source/backend abstraction rather than adding provider conditionals throughout `app.py`.
3. Browser streaming is a separate operating mode; do not bolt Chromium into the Pygame render loop.
4. Video is optional. A global audio-only policy must prevent browser streaming from starting and must survive reboot/update.
5. Video providers are registry/configuration entries. The launcher and web admin must not require code changes for every compatible browser service.
6. A source switch must be explicit, observable, and reversible.
7. Existing devices must survive updates. Any system-level change needs both fresh-install setup and an idempotent migration.
8. Provider availability is capability-based, not promised by name. DRM/browser compatibility can change outside this project.
9. Keep credentials and cookies out of git. Browser profiles live only on the device.
10. The web admin must expose explicit application/device actions, never a general-purpose remote shell.
11. Local web administration is LAN-only by default. Password protection is optional, but when enabled it must be enforced consistently and stored securely.
12. Provider URLs/configuration must be validated and must never be interpreted as commands or privileged local resource paths.
13. Log failures and gather evidence before adding special-case workarounds.
14. Raspberry Pi 3 performance is a constraint, not an excuse for fragile global optimizations that hurt Pi 4/5.
15. Keep the normal music UI simple even as the underlying architecture becomes more capable.

## Suggested first implementation sequence

1. Fix repository ownership in installer/update paths.
2. Introduce `PlaybackBackend`, provider-neutral playback state, and `SourceManager` while keeping Spotify as the only backend.
3. Add Bluetooth receiver mode.
4. Add AirPlay audio.
5. Add internet radio/local media.
6. Introduce the shared settings/device-control layer needed by both touchscreen and future web administration.
7. Add the local web admin with status, optional password protection, video lock, provider management, settings, update, restart, and shutdown controls.
8. Add the provider registry plus Streaming launcher and kiosk-mode lifecycle.
9. Validate YouTube first because it is useful without relying on the full commercial DRM matrix.
10. Validate Netflix, Prime Video, Disney+, and WOW individually on Pi 3/4/5.
11. Add parental controls and hardening after the technical path is proven.

## Documentation ownership

Update this roadmap whenever a phase changes materially, a provider is validated/rejected, or the architecture changes. Completed work should be checked off only after it is merged and verified on representative Raspberry Pi hardware.