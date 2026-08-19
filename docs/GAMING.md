# Gaming Mode Roadmap

This document extends the main media roadmap with an optional Gaming layer. Gaming is a separate foreground mode alongside Audio and Video.

The box acts as a thin client: games run on a local gaming PC or cloud/remote-PC service, while the Raspberry Pi handles video/audio streaming and controller/input forwarding.

## Product model

```text
Malti Box
   |
   +-- Audio
   |     +-- Spotify
   |     +-- Bluetooth Receiver
   |     +-- AirPlay
   |     +-- Radio / Local Media
   |
   +-- Video
   |     +-- Netflix / Disney+ / Prime / WOW / YouTube / custom
   |
   +-- Gaming
         +-- Steam Link
         +-- Shadow PC
         +-- GeForce NOW
         +-- future providers
```

Audio, Video, and Gaming are independently configurable. A device may run audio-only, audio+video, audio+gaming, or all layers.

## Global Gaming policy

- [ ] Add persistent `gaming_enabled` state.
- [ ] Hide the Gaming launcher when Gaming is disabled.
- [ ] Reject Gaming launch requests from touchscreen, Web Admin, API, or internal paths while disabled.
- [ ] Do not keep Gaming clients running in the background while Gaming is disabled.
- [ ] If Gaming is disabled remotely during a session, close the client safely and return to Mello.
- [ ] Persist the policy across reboot and updates.

## Mode architecture

Gaming is an exclusive foreground mode and must not be embedded inside the Pygame render loop.

```text
ModeManager
   |
   +-- AudioMode
   +-- VideoMode
   +-- GamingMode
```

Video and Gaming should reuse the same low-level display/audio foreground handoff where practical.

Proposed lifecycle:

```text
Mello/Pygame
    -> check gaming_enabled
    -> choose enabled Gaming provider
    -> pause active audio source
    -> save state
    -> release display resources
    -> prepare audio and controller routing
    -> start native client OR kiosk browser
    -> run session
    -> exit via Home/Back/system action
    -> stop client
    -> restore display/audio/input
    -> return to Mello
```

## Gaming provider model

Gaming providers may be `native` or `browser`.

Suggested fields:

```text
id
name
enabled
order
launch_type        native | browser
adapter_id         allow-listed native adapter
start_url          browser providers only
icon/artwork
browser_profile
compatibility      validated | experimental | unsupported | untested
hardware_notes
```

Security rule: owner-created custom Gaming providers may be browser providers only. The Web Admin must never accept arbitrary executable paths, commands, shell arguments, or systemd units. Native providers are explicit project adapters.

## Steam Link

Steam Link is the first native Gaming provider.

- [ ] Add a `SteamLinkAdapter` or equivalent.
- [ ] Install the native client through fresh-install setup plus `pi/migrate.sh` for existing boxes.
- [ ] Detect client availability before showing it as usable.
- [ ] Target Raspberry Pi 3 or newer where the current client/OS combination is validated.
- [ ] Prefer Ethernet for latency and reliability.
- [ ] Preserve Steam Link pairing/configuration.
- [ ] Validate controller handling and reliable return to Mello.

Steam Link remains usable even if Video is globally disabled because `gaming_enabled` and `video_enabled` are independent.

## Shadow PC

Prefer Shadow's official Raspberry Pi ARM64 client on supported hardware.

### Native Shadow

- [ ] Add a `ShadowPcAdapter` for the official native client.
- [ ] Target Raspberry Pi 4/5 unless Shadow changes its supported hardware policy.
- [ ] Detect supported hardware and installed client before enabling the native provider.
- [ ] Let Shadow manage its own login/session/2FA data; do not collect Shadow credentials in Mello.
- [ ] Preserve native-client session data locally where supported.
- [ ] Validate controller, keyboard, mouse, audio, 720p/60, display rotation, and exit/recovery.

### Shadow Browser fallback

- [ ] Keep browser Shadow as a separate optional fallback capability.
- [ ] Use an isolated browser profile.
- [ ] Let Shadow's web client own login/authentication.
- [ ] Mark browser Shadow separately as experimental/validated/unsupported.
- [ ] Do not assume browser is better solely because login is web-based; native is preferred when supported.

## GeForce NOW

GeForce NOW belongs in Gaming rather than the general Video provider list.

- [ ] Add GeForce NOW as a built-in Gaming provider preset.
- [ ] Treat Raspberry Pi browser use as experimental until validated on real hardware.
- [ ] If the service works with the installed Chromium/browser stack, launch it in a dedicated kiosk/browser profile.
- [ ] Validate login, game launch, gamepad input, fullscreen, audio, codec decode, latency, and exit/recovery.
- [ ] Do not build unsupported-device, user-agent, or DRM bypasses into the product.
- [ ] If NVIDIA later provides a supported Raspberry Pi/ARM64 native client, prefer an explicit native adapter after validation.

## Controller and input layer

- [ ] Add shared controller/input management for all Gaming providers.
- [ ] Support USB gamepads.
- [ ] Support Bluetooth gamepads where Linux supports them.
- [ ] Distinguish game controllers from Bluetooth audio devices.
- [ ] Show controller state in Gaming UI and Web Admin.
- [ ] Allow safe pairing/disconnect from Web Admin where practical.
- [ ] Preserve keyboard/mouse support for PC-style services such as Shadow.
- [ ] Validate mappings and reconnect behavior for representative Xbox/PlayStation/common controllers.

## Gaming launcher

```text
Gaming

[ Steam Link ]
[ Shadow PC ]
[ GeForce NOW ]
[ Custom browser provider ]
```

Only individually enabled and currently available providers are shown.

- [ ] Render from Gaming provider definitions rather than hard-coded visibility.
- [ ] Show immediate pressed/loading state.
- [ ] Show clear unavailable/offline feedback.
- [ ] Provide a system-level Home/Back escape path.
- [ ] Recover to Mello automatically after client crashes.

## Web Admin

Add a dedicated Gaming section.

- [ ] Global Gaming enable/disable.
- [ ] Enable/disable every Gaming provider individually.
- [ ] Reorder Gaming providers.
- [ ] Show launch type (`native` / `browser`).
- [ ] Show compatibility/availability status.
- [ ] Show whether required native clients are installed.
- [ ] Show controller state and safe pairing controls.
- [ ] Allow supported quality/resolution defaults where an adapter exposes them.
- [ ] Allow custom browser-based Gaming providers using validated HTTPS URLs.
- [ ] Never expose provider passwords/tokens/cookies/session secrets.
- [ ] Never allow arbitrary native executable definitions.

All per-service visibility and enable/disable behavior also follows `docs/SERVICE-MANAGEMENT.md`.

## Hardware policy

### Raspberry Pi 3

- Audio remains baseline.
- Steam Link can be targeted where validated.
- Native Shadow is not a target unless Shadow expands support.
- GeForce NOW browser remains experimental/unverified.

### Raspberry Pi 4

- Target Steam Link.
- Target official native Shadow client.
- Validate GeForce NOW browser experimentally.
- Prefer Ethernet for Gaming.

### Raspberry Pi 5

- Preferred target for broad Gaming capability.
- Target Steam Link and native Shadow.
- Validate GeForce NOW browser experimentally.
- Reuse the same foreground compositor/display strategy as Video where possible.

## Initial validation matrix

| Provider | Launch type | Pi 3 | Pi 4 | Pi 5 |
|---|---|---|---|---|
| Steam Link | native | target | target | target |
| Shadow PC | native | not target | target | target |
| Shadow Browser | browser | experimental | experimental | experimental |
| GeForce NOW | browser | experimental | experimental | experimental |
| Custom Gaming URL | browser | unverified | unverified | unverified |

## Exit criteria

Gaming is ready when:

1. Gaming can be enabled/disabled independently of Video.
2. Every Gaming provider can be individually enabled/disabled from Web Admin.
3. Steam Link launches and returns reliably on validated hardware.
4. Native Shadow launches and returns reliably on supported Pi 4/5 devices.
5. GeForce NOW has a documented tested status rather than an assumption.
6. Controllers survive provider launches and return cleanly to the system.
7. Custom browser Gaming providers can be added without arbitrary command execution.
8. Failed launches/crashes recover to Mello without a reboot.
9. System-level changes include fresh-install setup and matching idempotent migrations.

## Implementation order

1. Introduce shared foreground `ModeManager`/handoff.
2. Add persistent `gaming_enabled` and Gaming launcher shell.
3. Implement shared service enablement model from `docs/SERVICE-MANAGEMENT.md`.
4. Add controller/input service and Web Admin status.
5. Implement Steam Link native adapter.
6. Implement native Shadow adapter for Pi 4/5.
7. Add Shadow browser fallback only if useful after native validation.
8. Validate GeForce NOW experimentally.
9. Add safe custom browser Gaming providers.
10. Harden recovery, logging, migrations, and hardware compatibility documentation.
