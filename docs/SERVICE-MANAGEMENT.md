# Service and Feature Management

This document defines how individual Audio, Video, and Gaming services are enabled, disabled, shown, and managed from the local Web Admin.

The goal is simple: the box should only show and run the services that are actually wanted on that device.

## Core rule

Every optional user-facing service must have an independent persisted `enabled` state in addition to any higher-level mode switch.

Examples:

```text
Audio
  Spotify              enabled
  Bluetooth Receiver   enabled
  AirPlay              disabled
  Internet Radio       disabled
  Local Media          enabled

Video                  globally enabled
  Netflix              enabled
  Disney+              disabled
  Prime Video          enabled
  WOW                  disabled
  YouTube              enabled
  Custom provider      disabled

Gaming                 globally enabled
  Steam Link           enabled
  Shadow PC            disabled
  GeForce NOW          enabled
  Custom gaming URL    disabled
```

A global mode switch and an individual service switch are separate controls.

For example:

- `video_enabled = false` disables the whole Video layer regardless of individual provider flags.
- `video_enabled = true` allows only Video providers whose own `enabled` flag is true.
- `gaming_enabled = false` disables the whole Gaming layer regardless of individual Gaming-provider flags.
- `gaming_enabled = true` allows only enabled Gaming providers.

Audio does not need one mandatory global kill switch, but each optional Audio source must still be individually controllable. A future `audio_sources_enabled` policy may be added if there is a concrete product need.

## Web Admin

Add a dedicated `Services` or equivalent configuration page that gives the owner one place to control visibility and availability.

Suggested structure:

```text
Services

Audio
  [on ] Spotify
  [on ] Bluetooth Receiver
  [off] AirPlay
  [off] Internet Radio
  [on ] Local Media

Video
  [on ] Video mode
  [on ] Netflix
  [off] Disney+
  [on ] Prime Video
  [off] WOW
  [on ] YouTube

Gaming
  [on ] Gaming mode
  [on ] Steam Link
  [off] Shadow PC
  [on ] GeForce NOW
```

The exact UI can differ, but the owner must be able to understand at a glance which capabilities are active.

## Required behavior

- [ ] Every optional Audio source has a persisted `enabled` state.
- [ ] Every Video provider has a persisted `enabled` state.
- [ ] Every Gaming provider has a persisted `enabled` state.
- [ ] Global Video and Gaming switches remain separate from individual provider switches.
- [ ] Disabled services disappear from the child-facing launcher/navigation immediately where practical.
- [ ] Disabled services cannot be started through touchscreen, Web Admin, API, stale UI state, deep-link, or internal fallback path.
- [ ] If a service is disabled while it is currently active, stop/close it safely and switch back to an allowed/default state.
- [ ] If a provider is re-enabled, its previous device-local configuration/login/profile should remain intact unless the owner explicitly deletes/reset it.
- [ ] Service enablement state survives reboot and normal updates.
- [ ] Service settings use the same shared settings/service registry for touchscreen and Web Admin; do not duplicate state.

## UI visibility

Disabled services should normally not be shown to the normal user at all.

Examples:

- AirPlay disabled -> no AirPlay entry/status in the normal source selector.
- Netflix disabled -> no Netflix tile in Streaming.
- Gaming enabled but Shadow disabled -> Gaming remains visible, but Shadow does not appear.
- All Video providers disabled -> hide the Video/Streaming launcher even if `video_enabled` is technically true, or show a parent/admin-only empty-state only where useful.
- All Gaming providers disabled -> hide the Gaming launcher even if `gaming_enabled` is true.

The Web Admin must always continue to show disabled services so the owner can re-enable them.

## Runtime/service lifecycle

Where practical, disabling a service should also avoid unnecessary background work.

- [ ] Do not start provider-specific browser processes for disabled Video/Gaming providers.
- [ ] Do not start native Gaming clients for disabled providers.
- [ ] If an optional background daemon exists only for one disabled service, stop/disable it when safe and restart it when that service is re-enabled.
- [ ] Do not aggressively stop shared infrastructure such as PipeWire, Bluetooth, network management, or the Web Admin merely because one dependent service is disabled.
- [ ] Distinguish `hidden/disabled for product use` from `package uninstalled`; normal enable/disable should not repeatedly install/remove packages.
- [ ] Prefer lightweight service stop/start or runtime gating over package removal.

Examples:

```text
AirPlay disabled
  -> Shairport Sync may be stopped/disabled if no other feature needs it.

Steam Link disabled
  -> Steam Link remains installed, but cannot be launched and is hidden.

Netflix disabled
  -> Browser profile/cookies remain stored, but Netflix is not shown or launchable.
```

## Defaults and first setup

Initial defaults should stay conservative and simple.

- [ ] Existing Spotify behavior remains enabled after migration.
- [ ] Newly introduced optional services may default to disabled until configured/validated.
- [ ] Do not enable every new feature automatically on existing deployed boxes after an update.
- [ ] Preserve existing per-service choices during upgrades.
- [ ] Built-in provider presets can exist while disabled; adding a preset does not mean it must appear to the child/user.

## Service registry model

Use a provider/source registry or equivalent shared capability model instead of independent hard-coded booleans scattered throughout the app.

A service entry should be able to expose at least:

```text
id
category           audio | video | gaming
name
enabled
available          runtime/hardware availability
order
built_in
status             validated | experimental | unsupported | untested
requires_mode      optional: video | gaming
```

Provider-specific configuration belongs behind the provider/source adapter.

Important distinction:

- `enabled`: owner wants the service available.
- `available`: current hardware/software can actually run it.

A service may therefore be enabled in settings but unavailable on the current hardware. The normal launcher must not pretend it is usable; the Web Admin should explain the reason.

Example:

```text
Shadow PC
Owner setting: enabled
Hardware: Raspberry Pi 3
Availability: unavailable
Reason: native Shadow Raspberry Pi client targets Pi 4/5
```

## Admin actions

The Web Admin should support:

- [ ] enable/disable each service/provider
- [ ] enable/disable the global Video mode
- [ ] enable/disable the global Gaming mode
- [ ] reorder services/providers inside their launcher where applicable
- [ ] show availability and compatibility status
- [ ] preserve provider-specific configuration when toggling off/on
- [ ] reset/delete provider-specific data only through a separate explicit action
- [ ] apply most visibility changes without rebooting

## Security

- Service toggles are configuration mutations and must follow the Web Admin's authentication/CSRF rules.
- If optional admin password protection is enabled, service changes require an authenticated session.
- If the Web Admin is intentionally running without a password, the existing LAN-only warning applies.
- Never let service configuration become an arbitrary process/service launcher.
- Native provider IDs must map only to allow-listed project adapters.
- Custom Video/Gaming services may define validated browser URLs only; they cannot define shell commands or executable paths.

## Testing

Add tests for at least:

- persisted enabled/disabled state
- global mode + provider-state combinations
- launcher filtering
- disabled launch rejection
- disabling the currently active service
- re-enabling while preserving configuration
- unavailable hardware capability
- Web Admin synchronization with touchscreen state
- migration defaults for existing devices

## Exit criteria

Service management is complete when an owner can use the local Web Admin to reduce the box to exactly the desired feature set, and disabled services:

1. do not appear in the normal user interface,
2. cannot be launched through another code path,
3. do not consume avoidable provider-specific runtime resources,
4. keep their configuration for later re-enabling unless explicitly reset,
5. remain disabled after reboot/update,
6. can be re-enabled from the Web Admin without SSH or code changes.
