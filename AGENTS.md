# AGENTS.md

## Working style

- Act like a Staff Engineer. If an idea has a major technical downside, explain the concrete risk and ask whether it should still be attempted instead of silently implementing a fragile workaround.
- Use logs and observable evidence as much as possible; do not guess when the runtime can tell us what is happening.
- Do not ship band-aid solutions. Get to the root cause as much as possible.
- Explain important implementation decisions with real usage scenarios and clear logic rather than unnecessary jargon.

## Product principles

- The user must get direct feedback on every action. When moving to a different context/source, immediately show that the previous action stopped and the new content is loading. Button presses must highlight immediately rather than waiting for a network/system call.
- Keep UI elements limited and clean. New capabilities must not turn the normal music screen into a settings dashboard.
- Preserve the simple, kid-friendly interaction model even as the media architecture becomes more capable.
- Parent/system actions should stay behind deliberate interactions instead of adding permanent clutter to the main UI.
- Video is optional. The product must remain usable as a permanent audio-only streaming box even on hardware that supports browser video.
- A global video lock is a parent/admin policy. When disabled, Streaming/browser mode must not be launchable from the touchscreen or another internal code path.
- Gaming is optional and independently configurable from Video. A device may run audio-only, audio+video, audio+gaming, or all capabilities.
- A global gaming lock is a parent/admin policy. When disabled, native and browser Gaming providers must not be launchable from any UI/API path.

## Roadmap and architecture direction

- Read `docs/ROADMAP.md` before making architecture-level playback, audio-routing, browser, streaming, provider-registry, web-admin, Bluetooth, AirPlay, installer, or Raspberry Pi platform changes.
- Read `docs/GAMING.md` before making Gaming, controller/input, Steam Link, Shadow PC, GeForce NOW, foreground-app handoff, or Gaming-provider changes.
- Treat the roadmap documents as the current intended direction. If implementation reality requires changing them, update the relevant roadmap documentation in the same PR.
- The project is evolving from a Spotify-only player into a source-based media box with independent Audio, Video, and Gaming layers. Do not spread provider-specific conditionals throughout `mello/app.py`.
- New audio sources should be implemented behind a generic playback/source abstraction (`PlaybackBackend` / `SourceManager` or the equivalent architecture introduced by the roadmap).
- Spotify-specific concepts such as `spotify:*` URIs, go-librespot REST behavior, repeat-context rules, and Spotify session handling belong in the Spotify backend once the abstraction exists.
- Source changes must be explicit and safe: pause/stop the old source, update active-source state, route audio, then activate the new source. Avoid multiple sources competing for the same audio device.
- Browser video streaming is a separate operating mode from Pygame/Mello. Prefer a clean lifecycle that releases the display, starts a kiosk browser/compositor, and restores Mello afterward. Do not embed Chromium directly into the Pygame render loop without intentionally revisiting the architecture.
- Streaming mode must check the persisted global video policy before launching. If video is disabled while a browser session is active, close it safely and return to audio/Mello mode.
- Video providers must be data-driven through a `ProviderRegistry` or equivalent configuration layer. Do not add a new hard-coded UI branch or launcher path for every service.
- Built-in providers such as YouTube, Netflix, Prime Video, Disney+, and WOW should be presets in the registry, while compatible additional providers can be created from the web admin.
- Provider definitions should support a stable ID, display name, launch URL, enabled state, order, optional icon/artwork, browser-profile identifier, and compatibility notes where needed.
- Provider add/edit/delete/reorder operations must update the same registry used by the touchscreen launcher.
- Validate provider URLs server-side. Allow intended web schemes only; never treat provider URLs or configuration as commands or privileged local resource paths.
- Each provider should use an isolated persistent browser profile so logins/cookies can survive without unnecessarily sharing session state between services.
- User-added providers are unverified by default. Never imply that adding a provider means DRM, playback, hardware acceleration, or touch support is guaranteed.
- Commercial video services should use their own browser players. Do not reimplement provider players, bypass DRM, or add DRM-circumvention workarounds.
- Browser credentials, cookies, tokens, and user account data must remain device-local and must never be committed to git.

## Gaming mode

- Gaming must be an exclusive foreground mode alongside Audio and Video, coordinated by a single `ModeManager`/foreground-mode boundary or equivalent architecture.
- Reuse the same low-level display/audio foreground-app handoff for Video and Gaming where practical instead of maintaining two unrelated takeover mechanisms.
- New Gaming providers must declare a launch type: `native` or `browser`.
- Native Gaming providers are explicit allow-listed adapters implemented by the project. Never let the Web Admin define arbitrary executables, shell commands, systemd units, command-line arguments, or local executable paths.
- Custom owner-added Gaming providers may be browser providers only and must use the same server-side URL validation/security rules as custom Video providers.
- Steam Link should use a native adapter and is the first target Gaming provider.
- Shadow PC should prefer Shadow's official Raspberry Pi ARM64 client on supported hardware. Browser Shadow is a fallback/secondary capability, not the default merely because login occurs in a browser.
- GeForce NOW on Raspberry Pi is experimental until validated against the current NVIDIA-supported client/browser matrix. Do not bypass unsupported-device checks or depend on user-agent/DRM workarounds.
- Gaming launch must check persisted `gaming_enabled` state. If Gaming is disabled remotely while a session is active, close the client safely and return to Mello.
- Controller/input management should be shared across Gaming providers and must not corrupt Bluetooth audio receiver/output roles.
- Keep provider authentication inside the provider/native client or isolated browser profile. Never collect Steam, Shadow, NVIDIA, or other gaming-service passwords in Mello settings.
- Every native Gaming client must have an explicit install-detection, launch, exit/recovery, and capability-check path.
- Third-party client crashes must recover to Mello without requiring a reboot wherever practical.

## Local web administration

- The roadmap includes a local web administration interface reachable through the device hostname/IP on the local network.
- Web settings and touchscreen settings must use one shared settings/device-control layer. Do not create two independent sources of truth.
- The web admin may expose explicit controls such as video enable/disable, provider management, Gaming enable/disable/provider management, controller status, audio settings, network/Bluetooth configuration, update, restart, shutdown, and device status.
- Provider management must allow add, edit, enable/disable, reorder, remove, and restore built-in presets without requiring code changes where the provider type is safely configurable.
- Web-admin HTTP handlers must never accept or construct arbitrary shell commands from request input. Privileged operations must be implemented as explicit, allow-listed application/device-control actions.
- Keep read-only status APIs separate from privileged mutation APIs where practical.
- Admin password protection is optional. The owner must be able to enable it from the admin area, change the password later, or remove password protection again without SSH.
- When no admin password is configured, keep the interface LAN-only and clearly show that local network users may be able to administer the device.
- When a password is configured, require authentication before entering the admin interface and before configuration/system actions. Enforce that protection consistently across all admin routes.
- Store only a modern salted password hash; never store or log the plaintext password.
- Use session-based authentication with logout and reasonable expiry when password protection is enabled. Rate-limit failed login attempts.
- Removing password protection must require an authenticated session and an explicit confirmation while protection is currently enabled.
- Protect state-changing requests against CSRF and accidental repeated actions regardless of whether password protection is enabled, and rate-limit security-sensitive control paths where practical.
- Require explicit confirmation for disruptive actions such as update, restart, shutdown, reset, network changes, and destructive provider changes where appropriate.
- Bind/expose the admin interface only to the local/LAN environment by default. Do not intentionally publish it to the internet as part of setup.
- Never expose Spotify credentials, streaming-provider cookies/tokens, Gaming-provider credentials/session tokens, Widevine/provider sessions, passwords, or other sensitive account data through the web UI/API or logs.
- The web-admin service should remain available independently of Pygame/browser/native-Gaming mode where practical so a parent can recover or manage the device remotely on the LAN.

## Hardware support policy

- Raspberry Pi 3 remains the baseline supported device for the core audio experience (Spotify, Bluetooth audio, AirPlay/audio sources where validated, radio, and local media).
- Browser/video streaming on Raspberry Pi 3 is experimental. Optimize it by stopping or suspending unneeded services during Streaming mode instead of assuming Mello, go-librespot, and the browser can all remain fully active.
- Raspberry Pi 4 or newer is the target for a supported browser/video experience; Raspberry Pi 5 is the preferred target for responsiveness and headroom.
- Steam Link can target Raspberry Pi 3 or newer where the current package/runtime is validated.
- Shadow native Gaming targets Raspberry Pi 4/5 unless Shadow expands official hardware support.
- GeForce NOW on Raspberry Pi must remain marked experimental/unverified until real-device testing establishes a reliable path.
- Do not silently remove Pi 3 support while implementing Pi 4/5 features. Use runtime capability detection or clear feature gating where hardware differs.
- Audio-only mode must remain supported on Pi 4/5. More capable hardware must not force video or Gaming to be enabled.
- The physical display is 720x1280 and the normal user view is landscape. Browser/streaming/Gaming work must preserve the same perceived orientation and touch/input mapping.

## Existing devices and migrations

- This app runs on Raspberry Pi devices that auto-update from git. Always consider devices already in the field.
- If you add a Python dependency to `requirements.txt`, it is installed on the next auto-update.
- If you change system configuration or system dependencies — including apt packages, sudoers, systemd units, udev, boot/display configuration, PipeWire/WirePlumber, Bluetooth profiles, browsers/compositors, DRM packages, native Gaming clients, controller/input packages, web-admin services, firewall/network binding, or service permissions — you MUST add an idempotent migration in `pi/migrate.sh` as well as updating the fresh-install setup where applicable.
- If persisted provider configuration or other settings schemas change, add safe schema migration/version handling so existing custom providers and settings are preserved.
- Auto-update runs migrations after pulling code. Without a migration, existing devices can diverge from fresh installs and break.
- Migrations must be safe to run once on partially configured systems and should log what they changed.
- When adding a new service or native client, verify fresh install, migration from an existing install, enable/start behavior, restart behavior, exit/recovery behavior, and removal/rollback behavior where relevant.

## Repository ownership and updates

- This fork is intended to become the source of truth for its own installations. New work must not introduce installer/update URLs that point back to another repository unless explicitly required.
- When touching install/update logic, audit `install.sh`, `pi/setup.sh`, `pi/auto-update.sh`, documentation examples, and migrations together so fresh installs and existing devices follow the same repository.

## Testing and observability

- Add or update tests for logic changes, especially playback state, source switching, stale asynchronous commands, capability detection, video-policy enforcement, gaming-policy enforcement, mode switching, provider-registry CRUD/order/validation, optional web-admin authentication, privileged-action validation, and recovery behavior.
- For hardware/system changes, document and perform the most relevant real-device checks because unit tests cannot prove audio routing, DRM, display ownership, controller mapping, native client behavior, Bluetooth behavior, or network exposure.
- Log source switches, mode switches, audio-route changes, external service failures, browser-mode entry/exit, Gaming client entry/exit, provider configuration changes, video/gaming policy changes, web-admin system actions, migrations, and recovery paths with enough context to diagnose failures.
- Do not log passwords, cookies, authentication tokens, Spotify credentials, provider session data, Gaming account/session data, admin secrets, or other sensitive account information.
