# Service and Feature Management

This document defines how individual Audio, Video, and Gaming services are enabled, disabled, shown, and managed from the local Web Admin.

The box should only show and run services that are actually wanted on that device.

## Core rule

Every optional user-facing service has an independent persisted `enabled` state in addition to any higher-level mode switch.

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

Global mode switches and individual service switches are separate controls:

- `video_enabled = false` disables the entire Video layer regardless of provider flags.
- `video_enabled = true` allows only Video providers whose own `enabled` flag is true.
- `gaming_enabled = false` disables the entire Gaming layer regardless of provider flags.
- `gaming_enabled = true` allows only enabled Gaming providers.
- Audio sources such as Spotify, Bluetooth Receiver, AirPlay, Radio, and Local Media remain individually controllable.

## Web Admin

Add a `Services` page or equivalent central configuration view.

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

The owner must be able to understand at a glance which capabilities are active.

## Required behavior

- [ ] Every optional Audio source has a persisted `enabled` state.
- [ ] Every Video provider has a persisted `enabled` state.
- [ ] Every Gaming provider has a persisted `enabled` state.
- [ ] Global Video and Gaming switches remain separate from individual provider switches.
- [ ] Disabled services disappear from the normal user-facing launcher/navigation immediately where practical.
- [ ] Disabled services cannot be started through touchscreen, Web Admin, API, stale UI state, deep-link, or an internal fallback path.
- [ ] If a service is disabled while currently active, stop/close it safely and return to an allowed/default state.
- [ ] Re-enabling a service preserves its previous device-local configuration/login/profile unless the owner explicitly resets it.
- [ ] Service enablement state survives reboot and normal updates.
- [ ] Touchscreen and Web Admin use the same shared settings/service registry.

## UI visibility

Disabled services normally do not appear in the child-facing UI.

Examples:

- AirPlay disabled -> no AirPlay entry/status in the normal source selector.
- Netflix disabled -> no Netflix tile in Streaming.
- Gaming enabled but Shadow disabled -> Gaming remains visible, but Shadow is absent.
- All Video providers disabled -> hide the Video/Streaming launcher even if `video_enabled` is true, unless a parent/admin-only empty state is intentionally useful.
- All Gaming providers disabled -> hide the Gaming launcher even if `gaming_enabled` is true.

The Web Admin always continues to show disabled services so they can be re-enabled.

## Runtime/service lifecycle

Where practical, disabling a service should also avoid unnecessary background work.

- [ ] Do not start provider-specific browser processes for disabled Video/Gaming providers.
- [ ] Do not start native Gaming clients for disabled providers.
- [ ] If an optional daemon exists only for one disabled service, stop/disable it when safe and restart it when that service is re-enabled.
- [ ] Do not stop shared infrastructure such as PipeWire, Bluetooth, network management, or the Web Admin merely because one dependent service is disabled.
- [ ] Treat `disabled` separately from `uninstalled`; normal toggling must not repeatedly install/remove packages.

Examples:

```text
AirPlay disabled
  -> Shairport Sync may be stopped if nothing else needs it.

Steam Link disabled
  -> client remains installed but hidden and non-launchable.

Netflix disabled
  -> browser profile/cookies remain stored but Netflix is hidden and non-launchable.
```

## Defaults and migration

- [ ] Existing Spotify behavior stays enabled after migration.
- [ ] Newly introduced optional services may default to disabled until configured/validated.
- [ ] Do not automatically enable every new feature on already deployed boxes after an update.
- [ ] Preserve existing per-service choices during upgrades.
- [ ] Built-in provider presets may exist while disabled.

## Shared service registry

Prefer a shared capability/service registry over hard-coded booleans scattered through the application.

A service entry should expose at least:

```text
id
category           audio | video | gaming
name
enabled
available          current hardware/software availability
order
built_in
status             validated | experimental | unsupported | untested
requires_mode      optional: video | gaming
```

Important distinction:

- `enabled`: the owner wants the service available.
- `available`: the current device can actually run it.

Example:

```text
Shadow PC
Owner setting: enabled
Hardware: Raspberry Pi 3
Availability: unavailable
Reason: native Shadow Raspberry Pi client targets Pi 4/5
```

The normal launcher must not pretend an unavailable service can run. The Web Admin should explain why it is unavailable.

## Admin actions

- [ ] Enable/disable every service/provider individually.
- [ ] Enable/disable the global Video mode.
- [ ] Enable/disable the global Gaming mode.
- [ ] Reorder providers/services inside their launcher where applicable.
- [ ] Show availability and compatibility status.
- [ ] Preserve provider configuration while toggled off.
- [ ] Reset/delete provider-specific data only through a separate explicit action.
- [ ] Apply most visibility changes without rebooting.

## Security

- Service toggles are configuration mutations and follow the Web Admin authentication/CSRF rules.
- When optional admin password protection is enabled, service changes require authentication.
- When the Web Admin intentionally has no password, the existing LAN-only warning applies.
- Never turn service configuration into an arbitrary process launcher.
- Native provider IDs map only to allow-listed project adapters.
- Custom Video/Gaming services may define validated browser URLs only; never shell commands or executable paths.

## Testing

Add tests for:

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

Service management is complete when an owner can reduce the box to exactly the desired feature set and disabled services:

1. do not appear in the normal user interface,
2. cannot be launched through another code path,
3. do not consume avoidable provider-specific runtime resources,
4. keep their configuration for later re-enabling unless explicitly reset,
5. remain disabled after reboot/update,
6. can be re-enabled from the Web Admin without SSH or code changes.
