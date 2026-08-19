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
- Gaming is optional and independent from Video. The product must support audio-only, audio+video, audio+gaming, and full Audio+Video+Gaming configurations.
- Every optional Audio source, Video provider, and Gaming provider must be individually enableable/disableable from the shared settings/Web Admin layer.
- Disabled services must normally disappear from the normal user UI and must not remain launchable through alternate code paths.
- Video and Gaming time controls are optional parent/admin policies. Daily budgets and weekly schedules must each be independently enableable/disableable for Video and Gaming.

## Roadmap and architecture direction

- Read `docs/ROADMAP.md` before making architecture-level playback, audio-routing, browser, streaming, provider-registry, web-admin, Bluetooth, AirPlay, installer, or Raspberry Pi platform changes.
- Read `docs/GAMING.md` before Gaming, Steam Link, Shadow PC, GeForce NOW, controller/input, or foreground-app handoff changes.
- Read `docs/SERVICE-MANAGEMENT.md` before changing feature visibility, service enablement, provider availability, service lifecycle, or Web Admin service toggles.
- Read `docs/USAGE-LIMITS.md` before changing Video/Gaming daily budgets, weekday/time schedules, usage accounting, time-policy enforcement, timezone handling, or temporary parent overrides.
- Treat the roadmap documents as the current intended direction. If implementation reality requires changing them, update the relevant documentation in the same PR.
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

## Service and feature management

- Use one shared service/capability registry or equivalent source of truth for Audio, Video, and Gaming enablement instead of unrelated booleans scattered through the UI and managers.
- Distinguish `enabled` (the owner wants the service) from `available` (the current hardware/software can run it).
- Every optional Audio source such as Spotify, Bluetooth Receiver, AirPlay, Radio, and Local Media must have an independent persisted enabled state where applicable.
- Every Video provider and every Gaming provider must have an independent persisted enabled state in addition to the global `video_enabled` / `gaming_enabled` mode switches.
- Disabled services must be filtered from the normal launcher/source selector and rejected by touchscreen, Web Admin, API, stale/deep-link, and internal fallback launch paths.
- If a currently active service is disabled remotely, stop/close it safely and return to an allowed/default state.
- Re-enabling a service should preserve its device-local profile/configuration unless the owner explicitly resets/deletes that data.
- Do not automatically enable newly introduced optional features on existing devices after an update. Preserve existing service choices during migrations.
- Avoid unnecessary runtime work for disabled services, but do not stop shared infrastructure such as PipeWire, Bluetooth, NetworkManager, or the Web Admin just because one dependent feature is disabled.

## Usage limits and schedules

- Video and Gaming each have two independent optional time-policy controls: `daily_limit_enabled` and `schedule_enabled`.
- The owner may use only a daily budget, only a weekly schedule, both together, or neither. If both are disabled for a category, time policy must not restrict that category at all.
- Turning a daily budget or weekly schedule off must preserve its configured values for later re-enabling.
- Effective launch permission combines global mode state, per-provider enabled state, hardware/software availability, any enabled weekly schedule, and any enabled daily budget. The strictest **enabled** rule wins.
- A disabled daily budget means unlimited use from the quota perspective; an enabled schedule may still restrict access.
- A disabled weekly schedule means unrestricted weekdays/times from the schedule perspective; an enabled daily budget may still restrict access.
- Time policy must be enforced by the shared mode/device-control layer rather than trusting provider-specific playback state or UI.
- By default, counted Video usage is foreground time owned by `VideoMode`, and counted Gaming usage is foreground time owned by `GamingMode`. Do not rely on Netflix, Steam, Shadow, GeForce NOW, or another provider to report reliable pause/play state.
- While a category's daily budget is disabled, usage must not be deducted from that daily quota. Previously recorded usage for the day must be preserved so toggling the switch is not a reset mechanism.
- Persist counted usage regularly and on clean mode exit so reboot/power cycling does not trivially bypass an enabled daily budget.
- Daily reset, weekday calculation, and schedule windows use the configured local device timezone. Handle invalid clocks, timezone changes, and DST deterministically.
- `time-blocked` is distinct from `disabled` and `unavailable`. The touchscreen and Web Admin should expose a clear reason plus remaining/next available time when relevant.
- When an active session reaches an enabled quota or schedule boundary, warn the user where practical, persist accounting, close the provider cleanly, and return to Mello. Never power off the Raspberry Pi to enforce a limit.
- If an Admin turns a time rule off while a session is running, stop enforcing that specific rule immediately without terminating an otherwise allowed session.
- If an Admin enables a rule while a session is running and the current session violates the new rule, apply the new policy consistently and return to Mello after clear feedback.
- Changing/toggling a budget or schedule must not silently reset today's usage. `Reset today's usage` is a separate explicit privileged Admin action.
- Any temporary extra-time override must be explicit, parent/admin controlled, and time-bounded; never add a permanent child-facing bypass.

## Gaming mode

- Gaming is an exclusive foreground mode alongside Audio and Video, coordinated through a shared `ModeManager`/foreground-mode boundary or equivalent architecture.
- Reuse the same low-level display/audio handoff for Video and Gaming where practical.
- Gaming providers declare a launch type: `native` or `browser`.
- Native Gaming providers are explicit allow-listed adapters implemented by the project. Never allow Web Admin input to define arbitrary executable paths, shell commands, systemd units, or command lines.
- Owner-added custom Gaming providers may be browser providers only and must use the same validated URL security rules as custom Video providers.
- Steam Link should use a native adapter and is the first target Gaming provider.
- Shadow PC should prefer Shadow's official Raspberry Pi ARM64 native client on supported hardware; browser Shadow is a fallback/secondary capability.
- GeForce NOW on Raspberry Pi remains experimental until real-device validation proves a reliable path. Do not depend on unsupported-device, user-agent, or DRM bypasses.
- Gaming launch must check persisted `gaming_enabled`, per-provider enabled state, availability, and current Gaming time-policy result before launching.
- Controller/input management should be shared across Gaming providers and must not break Bluetooth audio receiver/output roles.
- Keep Steam, Shadow, NVIDIA, and other Gaming service authentication inside the native client or isolated browser profile; never collect those passwords in Mello settings.
- Third-party Gaming client crashes must recover to Mello without requiring a reboot where practical.

## Local web administration

- The roadmap includes a local web administration interface reachable through the device hostname/IP on the local network.
- Web settings and touchscreen settings must use one shared settings/device-control layer. Do not create two independent sources of truth.
- The web admin may expose explicit controls such as per-service enable/disable, video enable/disable, provider management, Gaming enable/disable/provider management, controller status, Video/Gaming daily budgets and weekly schedules, audio settings, network/Bluetooth configuration, update, restart, shutdown, and device status.
- Video and Gaming time settings must expose separate on/off switches for daily budget and weekly schedule. Do not force one rule to be enabled just because the other is used.
- The Web Admin should show the configured device timezone/time, current time-policy result, used/remaining daily budget when enabled, and next allowed time window when schedule enforcement is enabled.
- The Web Admin must continue to show disabled services so the owner can re-enable them even though they are hidden from the normal user interface.
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
- Require explicit confirmation for disruptive actions such as update, restart, shutdown, reset, network changes, destructive provider changes, and resetting today's usage where appropriate.
- Bind/expose the admin interface only to the local/LAN environment by default. Do not intentionally publish it to the internet as part of setup.
- Never expose Spotify credentials, streaming-provider cookies/tokens, Gaming-provider credentials/session data, Widevine/provider sessions, passwords, or other sensitive account data through the web UI/API or logs.
- The web-admin service should remain available independently of Pygame/browser/native-Gaming mode where practical so a parent can recover or manage the device remotely on the LAN.

## Hardware support policy

- Raspberry Pi 3 remains the baseline supported device for the core audio experience (Spotify, Bluetooth audio, AirPlay/audio sources where validated, radio, and local media).
- Browser/video streaming on Raspberry Pi 3 is experimental. Optimize it by stopping or suspending unneeded services during Streaming mode instead of assuming Mello, go-librespot, and the browser can all remain fully active.
- Raspberry Pi 4 or newer is the target for a supported browser/video experience; Raspberry Pi 5 is the preferred target for responsiveness and headroom.
- Steam Link can target Raspberry Pi 3 or newer where the current client/runtime is validated.
- Shadow native Gaming targets Raspberry Pi 4/5 unless Shadow expands official hardware support.
- GeForce NOW on Raspberry Pi remains experimental/unverified until real-device testing establishes a reliable path.
- Do not silently remove Pi 3 support while implementing Pi 4/5 features. Use runtime capability detection or clear feature gating where hardware differs.
- Audio-only mode must remain supported on Pi 4/5. More capable hardware must not force video or Gaming to be enabled.
- The physical display is 720x1280 and the normal user view is landscape. Browser/streaming/Gaming work must preserve the same perceived orientation and touch/input mapping.

## Existing devices and migrations

- This app runs on Raspberry Pi devices that auto-update from git. Always consider devices already in the field.
- If you add a Python dependency to `requirements.txt`, it is installed on the next auto-update.
- If you change system configuration or system dependencies — including apt packages, sudoers, systemd units, udev, boot/display configuration, PipeWire/WirePlumber, Bluetooth profiles, browsers/compositors, DRM packages, native Gaming clients, controller/input packages, web-admin services, firewall/network binding, or service permissions — you MUST add an idempotent migration in `pi/migrate.sh` as well as updating the fresh-install setup where applicable.
- If persisted provider/service/time-policy configuration or other settings schemas change, add safe schema migration/version handling so existing custom providers, enabled/disabled choices, daily-budget/schedule toggles, configured windows, usage accounting, and other settings are preserved.
- Auto-update runs migrations after pulling code. Without a migration, existing devices can diverge from fresh installs and break.
- Migrations must be safe to run once on partially configured systems and should log what they changed.
- When adding a new service or native client, verify fresh install, migration from an existing install, enable/start behavior, restart behavior, exit/recovery behavior, and removal/rollback behavior where relevant.

## Repository ownership and updates

- This fork is intended to become the source of truth for its own installations. New work must not introduce installer/update URLs that point back to another repository unless explicitly required.
- When touching install/update logic, audit `install.sh`, `pi/setup.sh`, `pi/auto-update.sh`, documentation examples, and migrations together so fresh installs and existing devices follow the same repository.

## Testing and observability

- Add or update tests for logic changes, especially playback state, source switching, service enablement/availability, launcher filtering, disabled launch rejection, Video/Gaming daily-budget and schedule toggles, combined time-policy decisions, usage persistence/reset behavior, time boundaries, timezone/DST handling, stale asynchronous commands, capability detection, video-policy enforcement, gaming-policy enforcement, mode switching, provider-registry CRUD/order/validation, optional web-admin authentication, privileged-action validation, and recovery behavior.
- For hardware/system changes, document and perform the most relevant real-device checks because unit tests cannot prove audio routing, DRM, display ownership, controller mapping, native client behavior, Bluetooth behavior, or network exposure.
- Log source switches, mode switches, audio-route changes, external service failures, browser-mode entry/exit, Gaming client entry/exit, provider/service configuration changes, time-policy changes/expiry events, video/gaming policy changes, web-admin system actions, migrations, and recovery paths with enough context to diagnose failures.
- Do not log passwords, cookies, authentication tokens, Spotify credentials, provider session data, Gaming account/session data, admin secrets, or other sensitive account information.
