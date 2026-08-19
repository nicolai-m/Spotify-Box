# Gaming Mode Roadmap

This document extends `docs/ROADMAP.md` with the planned Gaming layer. Gaming is a separate optional foreground mode alongside Audio and Video/Streaming.

The goal is not to run modern PC games locally on the Raspberry Pi. The box should act as a thin streaming client for a local gaming PC or a cloud-gaming/remote-PC service.

## Product model

The device should expose three independent capability layers:

```text
Mello / Malti Box
   |
   +-- Audio
   |     +-- Spotify
   |     +-- Bluetooth receiver
   |     +-- AirPlay
   |     +-- Radio/local media
   |
   +-- Video
   |     +-- browser kiosk providers
   |     +-- Netflix / Disney+ / Prime Video / WOW / YouTube / custom
   |
   +-- Gaming
         +-- Steam Link      (native)
         +-- Shadow PC       (native preferred, browser fallback)
         +-- GeForce NOW     (browser/experimental on Raspberry Pi)
         +-- future gaming providers
```

Audio, Video, and Gaming must be independently enableable. A deployment may therefore be configured as audio-only, audio+video, audio+gaming, or with all three layers enabled.

## Global Gaming policy

Introduce a persistent `gaming_enabled` policy separate from `video_enabled`.

- [ ] Add `gaming_enabled` to the shared device settings layer.
- [ ] Hide the Gaming launcher when Gaming is disabled.
- [ ] Reject all Gaming launch requests when Gaming is disabled, including web-admin/API requests.
- [ ] Do not keep gaming clients running in the background while Gaming is disabled.
- [ ] If Gaming is disabled remotely while a gaming session is active, close the client safely and return to Mello.
- [ ] Allow the policy to be changed from protected local settings and from the web administration interface.
- [ ] Persist the policy across reboot and normal updates.

## Mode architecture

Gaming must be treated as an exclusive foreground mode rather than being embedded inside the Pygame render loop.

Introduce a top-level mode boundary such as:

```text
ModeManager
   |
   +-- AudioMode
   +-- VideoMode
   +-- GamingMode
```

The exact names may change, but one component must own foreground-mode transitions and display/audio/input handoff.

Proposed Gaming lifecycle:

```text
Mello/Pygame
    -> check gaming_enabled
    -> user selects Gaming
    -> choose Gaming provider
    -> pause/stop active audio source
    -> persist Mello state
    -> release display resources
    -> prepare PipeWire/audio output
    -> prepare controller/input routing
    -> start native client OR kiosk browser
    -> run gaming session
    -> user exits through Home/Back/system action
    -> stop gaming client
    -> restore display/audio/input
    -> return to Mello
```

Video and Gaming should reuse the same low-level foreground application/display handoff wherever practical instead of implementing two unrelated ways to take ownership of the screen.

## Gaming provider model

Gaming providers have different launch requirements. Do not force every provider into a browser and do not allow arbitrary executable paths from the web admin.

Define two launch classes:

- `native`: an allow-listed application adapter implemented by the project.
- `browser`: a validated HTTPS URL opened with a provider-specific browser profile in kiosk mode.

Suggested provider data:

```text
id
name
enabled
order
launch_type        native | browser
adapter_id         allow-listed native adapter, if native
start_url          if browser
icon/artwork
browser_profile    if browser
compatibility      validated | experimental | unsupported | untested
hardware_notes
```

Security rule: custom providers created by the owner may define browser providers only. The web admin must never allow a user-supplied executable, command line, shell command, systemd unit, or arbitrary native application path. Native gaming providers are explicit project adapters.

## Steam Link

Steam Link is the first native Gaming provider.

- [ ] Add a `SteamLinkAdapter` or equivalent native-provider integration.
- [ ] Install Steam Link through fresh-install setup and an idempotent `pi/migrate.sh` migration when Gaming support is implemented.
- [ ] Support Raspberry Pi 3 or newer where the installed OS/client combination is validated.
- [ ] Prefer wired Ethernet for the best latency/reliability.
- [ ] Detect whether Steam Link is installed before advertising the provider as available.
- [ ] Add controller discovery/status before launch where practical.
- [ ] Preserve pairing/configuration created by the Steam Link client.
- [ ] Ensure exiting Steam Link reliably returns display, audio, and controller ownership to Mello.

Steam Link should remain usable even when browser Video streaming is disabled; `gaming_enabled` and `video_enabled` are independent policies.

## Shadow PC

Shadow should use its official Raspberry Pi client as the preferred launch path on supported hardware.

### Preferred native path

- [ ] Add an allow-listed `ShadowPcAdapter` for the official Raspberry Pi ARM64 client.
- [ ] Target Raspberry Pi 4/5 for the native Shadow client unless Shadow changes its supported hardware requirements.
- [ ] Detect client installation and supported hardware before enabling the native provider.
- [ ] Keep Shadow's own login/session data device-local.
- [ ] Allow the Shadow application to handle authentication, including any provider-side verification/2FA flows.
- [ ] Preserve Shadow application state across Mello restarts/updates where possible.
- [ ] Validate controller, keyboard, mouse, audio, 720p/60 and display rotation on the actual box.

The native client is preferred because Shadow explicitly supports a Raspberry Pi ARM64 application and the native application exposes capabilities that the browser client does not.

### Browser fallback

Shadow PC in Browser may be offered as an alternative/fallback provider when the browser path works on the installed Raspberry Pi/browser stack.

- [ ] Give the browser fallback its own isolated browser profile.
- [ ] Let the Shadow website/browser own login and account authentication; do not collect Shadow credentials in Mello.
- [ ] Treat browser Shadow as a separate capability from native Shadow so it can be independently marked experimental/unsupported.
- [ ] Do not assume browser mode is better merely because login occurs on a website; the native application can retain its own authenticated session.
- [ ] Account for browser-client feature limitations compared with the native application.

## GeForce NOW

GeForce NOW belongs in Gaming, not in the general Video provider list.

For Raspberry Pi, treat it as experimental until real-device validation proves a reliable supported path.

- [ ] Add a built-in GeForce NOW Gaming provider preset.
- [ ] Start with a dedicated Chromium/browser profile and kiosk/web-app-style launch only if the current NVIDIA service accepts the Raspberry Pi browser stack.
- [ ] Mark the provider `experimental` on Raspberry Pi until login, game launch, gamepad input, H.264/H.265 decode, audio, fullscreen, latency, and exit/recovery are verified.
- [ ] Do not hide unsupported-device checks or build user-agent/DRM bypasses as a production dependency.
- [ ] If NVIDIA later ships an ARM64 Raspberry Pi/native Linux client, prefer an explicit native adapter after validation rather than retaining a browser workaround by default.

NVIDIA's current native Linux desktop application targets Ubuntu 24.04 on x86/x64-class hardware, while its documented browser support names Windows, macOS and Chromebook. Raspberry Pi/ARM browser operation therefore must not be presented as officially supported before it has been validated.

## Controller and input layer

Gaming needs a shared controller/input service rather than provider-specific Bluetooth menus.

- [ ] Reuse the existing Bluetooth device manager where practical, but distinguish audio devices from controllers.
- [ ] Support USB gamepads.
- [ ] Support Bluetooth gamepads when the underlying Linux/input stack supports them.
- [ ] Show connected controller state in Gaming UI and Web Admin.
- [ ] Allow pairing/disconnect from the Web Admin where safe.
- [ ] Do not let Gaming controller routing break Bluetooth audio input/output roles.
- [ ] Preserve keyboard/mouse support for login and PC-style services such as Shadow.
- [ ] Test button mappings and disconnect/reconnect behavior for representative Xbox, PlayStation, and other common controllers.

## Gaming launcher UX

Keep the child-facing Gaming launcher intentionally small.

Example:

```text
Gaming

[ Steam Link ]
[ Shadow PC ]
[ GeForce NOW ]
[ ...future providers ]
```

The launcher should render from enabled Gaming provider definitions, but native providers must still map to allow-listed adapters rather than arbitrary commands.

- [ ] Show immediate pressed/loading state on launch.
- [ ] Show a simple unavailable/offline state instead of failing silently.
- [ ] Provide a system-level Home/Back escape path that a third-party client/browser cannot permanently hide.
- [ ] Restore Mello automatically after a gaming-client crash.

## Web administration

Add a dedicated Gaming section to the planned local Web Admin.

- [ ] Global Gaming enable/disable switch.
- [ ] Enable/disable individual built-in Gaming providers.
- [ ] Reorder Gaming providers.
- [ ] Show provider launch type (`native` / `browser`) and compatibility status.
- [ ] Show whether required native clients are installed.
- [ ] Show controller connection state and expose safe pairing controls.
- [ ] Show basic network readiness/latency guidance where practical.
- [ ] Allow safe Gaming defaults such as target resolution/quality where the provider adapter supports them.
- [ ] Do not expose provider passwords, tokens, cookies, or native-client session secrets.
- [ ] Allow custom browser-based Gaming providers to be added using the same HTTPS URL validation rules as Video providers, but keep them inside Gaming rather than the Video launcher.
- [ ] Never allow the Web Admin to define arbitrary native executable commands.

## Hardware policy

### Raspberry Pi 3

- Audio remains the baseline supported product.
- Steam Link is a valid Gaming target where the current Steam Link package works.
- Shadow native is not a target unless Shadow expands support.
- GeForce NOW browser is experimental/unverified.
- Keep Gaming optional so Pi 3 devices do not pay a runtime cost when the feature is disabled.

### Raspberry Pi 4

- Target Steam Link.
- Target the official Shadow Raspberry Pi client.
- Validate browser GeForce NOW experimentally.
- Prefer Ethernet for Gaming sessions.

### Raspberry Pi 5

- Preferred target for the broadest Gaming capability and UI responsiveness.
- Target Steam Link and official Shadow Raspberry Pi client.
- Validate browser GeForce NOW experimentally.
- Reuse the same foreground compositor/display strategy planned for browser Video where possible.

## Validation matrix

| Provider | Launch type | Pi 3 | Pi 4 | Pi 5 | Initial policy |
|---|---|---|---|---|---|
| Steam Link | native | target | target | target | supported after device validation |
| Shadow PC | native | unsupported target | target | target | native preferred |
| Shadow PC Browser | browser | experimental | experimental | experimental | fallback only |
| GeForce NOW | browser | experimental | experimental | experimental | do not claim official Pi support |
| Custom Gaming URL | browser | unverified | unverified | unverified | validate individually |

## Exit criteria

Gaming mode is ready when:

1. Gaming can be globally enabled or disabled independently of Video.
2. Steam Link can launch and return to Mello reliably on supported hardware.
3. Shadow's native Raspberry Pi client can launch and return reliably on Pi 4/5.
4. GeForce NOW has a clearly documented tested status rather than an assumption.
5. Controllers survive provider launches and return cleanly to the system.
6. Gaming providers can be enabled/disabled and ordered from the Web Admin.
7. Custom browser Gaming providers can be added safely without enabling arbitrary command execution.
8. Crashes or failed launches recover to Mello without requiring a reboot.
9. All new apt/systemd/permissions/display changes have matching `pi/migrate.sh` migrations for existing devices.

## Implementation order

1. Introduce `ModeManager`/foreground application handoff shared by Video and Gaming.
2. Add persistent `gaming_enabled` policy and Gaming launcher shell.
3. Add controller/input service and Web Admin status.
4. Implement Steam Link native adapter first.
5. Implement Shadow native adapter for Pi 4/5.
6. Add Shadow browser fallback only after native behavior is proven.
7. Experimentally validate GeForce NOW on the Raspberry Pi browser stack.
8. Add safe custom browser Gaming providers.
9. Add hardening, recovery, metrics, and real-device compatibility documentation.

Update this document when Valve, Shadow, NVIDIA, Raspberry Pi OS, browser support, or the selected display/compositor architecture materially changes.