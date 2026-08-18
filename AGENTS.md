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

## Roadmap and architecture direction

- Read `docs/ROADMAP.md` before making architecture-level playback, audio-routing, browser, streaming, Bluetooth, AirPlay, installer, or Raspberry Pi platform changes.
- Treat the roadmap as the current intended direction. If implementation reality requires changing it, update `docs/ROADMAP.md` in the same PR.
- The project is evolving from a Spotify-only player into a source-based media box. Do not spread new provider-specific conditionals throughout `mello/app.py`.
- New audio sources should be implemented behind a generic playback/source abstraction (`PlaybackBackend` / `SourceManager` or the equivalent architecture introduced by the roadmap).
- Spotify-specific concepts such as `spotify:*` URIs, go-librespot REST behavior, repeat-context rules, and Spotify session handling belong in the Spotify backend once the abstraction exists.
- Source changes must be explicit and safe: pause/stop the old source, update active-source state, route audio, then activate the new source. Avoid multiple sources competing for the same audio device.
- Browser video streaming is a separate operating mode from Pygame/Mello. Prefer a clean lifecycle that releases the display, starts a kiosk browser/compositor, and restores Mello afterward. Do not embed Chromium directly into the Pygame render loop without intentionally revisiting the architecture.
- Commercial video services should use their own browser players. Do not reimplement provider players, bypass DRM, or add DRM-circumvention workarounds.
- Treat provider support as capability-tested, not guaranteed. Netflix, Prime Video, Disney+, WOW, and similar services may change DRM/browser requirements outside this repository.
- Browser credentials, cookies, tokens, and user account data must remain device-local and must never be committed to git.

## Hardware support policy

- Raspberry Pi 3 remains the baseline supported device for the core audio experience (Spotify, Bluetooth audio, AirPlay/audio sources where validated, radio, and local media).
- Browser/video streaming on Raspberry Pi 3 is experimental. Optimize it by stopping or suspending unneeded services during Streaming mode instead of assuming Mello, go-librespot, and the browser can all remain fully active.
- Raspberry Pi 4 or newer is the target for a supported browser/video experience; Raspberry Pi 5 is the preferred target for responsiveness and headroom.
- Do not silently remove Pi 3 support while implementing Pi 4/5 features. Use runtime capability detection or clear feature gating where hardware differs.
- The physical display is 720x1280 and the normal user view is landscape. Browser/streaming work must preserve the same perceived orientation and touch mapping.

## Existing devices and migrations

- This app runs on Raspberry Pi devices that auto-update from git. Always consider devices already in the field.
- If you add a Python dependency to `requirements.txt`, it is installed on the next auto-update.
- If you change system configuration or system dependencies — including apt packages, sudoers, systemd units, udev, boot/display configuration, PipeWire/WirePlumber, Bluetooth profiles, browsers/compositors, DRM packages, or service permissions — you MUST add an idempotent migration in `pi/migrate.sh` as well as updating the fresh-install setup where applicable.
- Auto-update runs migrations after pulling code. Without a migration, existing devices can diverge from fresh installs and break.
- Migrations must be safe to run once on partially configured systems and should log what they changed.
- When adding a new service, verify fresh install, migration from an existing install, enable/start behavior, restart behavior, and removal/rollback behavior where relevant.

## Repository ownership and updates

- This fork is intended to become the source of truth for its own installations. New work must not introduce installer/update URLs that point back to another repository unless explicitly required.
- When touching install/update logic, audit `install.sh`, `pi/setup.sh`, `pi/auto-update.sh`, documentation examples, and migrations together so fresh installs and existing devices follow the same repository.

## Testing and observability

- Add or update tests for logic changes, especially playback state, source switching, stale asynchronous commands, capability detection, and recovery behavior.
- For hardware/system changes, document and perform the most relevant real-device checks because unit tests cannot prove audio routing, DRM, display ownership, or Bluetooth behavior.
- Log source switches, audio-route changes, external service failures, browser-mode entry/exit, migrations, and recovery paths with enough context to diagnose failures.
- Do not log passwords, cookies, authentication tokens, Spotify credentials, provider session data, or other sensitive account information.
