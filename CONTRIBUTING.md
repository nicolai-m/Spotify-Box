# Contributing to Mello

Thanks for your interest in contributing. Mello is currently a simple Spotify-focused music player for kids, and the roadmap expands it toward a multi-source media box without losing that simplicity.

Before architecture-level playback, audio-routing, browser/streaming, Bluetooth, AirPlay, installer, or Raspberry Pi platform work, read [docs/ROADMAP.md](docs/ROADMAP.md) and [AGENTS.md](AGENTS.md).

## Development Setup

You need a Raspberry Pi with Mello installed (see the [README](README.md)). Edit code on your machine, then sync and test on the Pi:

```bash
./dev-pi.sh --host user@host  # Syncs changes to Pi over SSH and streams logs
```

## Running Tests

```bash
pytest tests/ -v
```

## Making Changes

1. Fork the repo and create a branch from `main`
2. Check whether the change affects a roadmap phase or architecture rule
3. Make your changes
4. Run the tests: `pytest tests/ -v`
5. Test hardware/system changes on an appropriate Raspberry Pi
6. Update `docs/ROADMAP.md` when a phase, provider validation, or architecture decision changes materially
7. Open a pull request

### What makes a good PR

- **Small and focused** — one feature, architecture step, or fix per PR
- **Tested** — add or update tests for logic changes
- **Hardware-aware** — explain which Pi generation and physical audio/display path were tested when relevant
- **Migration-safe** — system-level changes include fresh-install setup and an idempotent `pi/migrate.sh` migration
- **Descriptive** — explain what changed, why, user impact, and relevant recovery/fallback behavior

## Architecture Rules

- Keep provider-specific behavior behind backend/source boundaries instead of adding provider checks throughout `mello/app.py`.
- New audio sources should integrate through the generic playback/source abstraction described in the roadmap.
- Keep source switching explicit: stop/pause the previous source, switch routing/state, then activate the new source.
- Treat browser video streaming as a separate operating mode that releases and restores the display cleanly.
- Do not embed Chromium into the Pygame render loop without a deliberate architecture change.
- Treat commercial streaming compatibility as something that must be validated on real hardware rather than assumed.
- Treat Raspberry Pi 3 as the supported audio baseline. Video on Pi 3 is experimental; Pi 4+ is the target for supported browser video.
- Preserve the simple kid-friendly main UI and immediate feedback for every action.

## Existing Device Compatibility

Devices auto-update from git, so system-level changes must work for both fresh installs and already-installed units.

If you change any of the following, update the corresponding fresh-install setup **and** add an idempotent migration in `pi/migrate.sh`:

- apt packages
- systemd units
- sudoers
- udev rules
- PipeWire/WirePlumber or Bluetooth profiles
- browser/compositor or DRM packages
- boot/display configuration
- service permissions or other machine-level configuration

Log migration actions clearly and avoid destructive assumptions about partially configured devices.

## Project Structure

```text
mello/
├── api/          # Spotify/catalog APIs; provider-specific APIs should stay isolated
├── handlers/     # Touch & event input
├── managers/     # Feature managers (sleep, carousel, Bluetooth, future sources/modes)
├── controllers/  # Playback/volume and provider-neutral control logic
├── ui/           # Pygame renderer & helpers
├── config.py     # Application constants/configuration
└── app.py        # Main application orchestration

pi/               # Fresh-install, migration, systemd and Raspberry Pi integration
docs/             # Roadmap and architecture/project documentation
```

As the playback abstraction is implemented, prefer focused backend/source modules over expanding the Spotify-specific API surface.

## Roadmap Validation

When completing roadmap items, do not check them off based on code alone. Verify the relevant behavior on representative hardware and document known limits, especially for:

- Bluetooth audio routing
- AirPlay/source arbitration
- display ownership and touch rotation
- kiosk browser lifecycle
- DRM playback
- memory/CPU/thermal behavior on Pi 3/4/5
- recovery after browser/service crashes
- updates from an existing installed device

## Questions?

Open an issue with the hardware model, relevant logs, expected behavior, and what was already tested.
