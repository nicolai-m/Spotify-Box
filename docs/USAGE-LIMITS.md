# Video and Gaming Usage Limits

This document defines parental time controls for the optional Video and Gaming layers. The controls are configured from the local Web Admin and enforced by the shared settings/device-control layer, not by individual provider pages.

The goal is to let an owner decide both **how long per day** Video/Gaming may be used and **at which times on which weekdays** those modes are available.

## Scope

Video and Gaming have independent time policies.

Examples:

```text
Video
  enabled: true
  daily limit: 60 minutes
  schedule:
    Monday-Friday: 15:00-19:00
    Saturday-Sunday: 09:00-20:00

Gaming
  enabled: true
  daily limit: 90 minutes
  schedule:
    Monday-Friday: 16:00-19:00
    Saturday: 10:00-21:00
    Sunday: 10:00-18:00
```

Audio is not time-limited by this feature unless a separate roadmap item explicitly adds Audio limits later.

## Core permission rule

A Video or Gaming provider may launch only when **all** relevant conditions are true:

```text
mode globally enabled
AND provider individually enabled
AND provider available on this hardware
AND current weekday/time is inside an allowed window
AND daily quota has remaining time
```

The strictest active rule always wins.

Examples:

- Gaming has 45 minutes remaining, but the allowed window ends in 10 minutes -> only 10 minutes are currently available.
- Video has no daily limit, but today's schedule is 15:00-18:00 -> it cannot launch outside that window.
- Steam Link is enabled, but `gaming_enabled = false` -> it cannot launch.
- Netflix is enabled and the Video window is open, but today's Video quota is exhausted -> it cannot launch.

## Web Admin UI

Add a dedicated `Time limits` / `Usage limits` section, or equivalent controls inside the Video and Gaming settings pages.

Suggested structure:

```text
Time limits

VIDEO
  Time control                 [on]
  Daily limit                  60 min
  Remaining today             37 min

  Monday       [on]  15:00 - 19:00
  Tuesday      [on]  15:00 - 19:00
  Wednesday    [on]  15:00 - 19:00
  Thursday     [on]  15:00 - 19:00
  Friday       [on]  15:00 - 20:00
  Saturday     [on]  09:00 - 20:00
  Sunday       [off]

GAMING
  Time control                 [on]
  Daily limit                  90 min
  Remaining today             62 min

  Monday       [on]  16:00 - 19:00
  Tuesday      [on]  16:00 - 19:00
  Wednesday    [on]  16:00 - 19:00
  Thursday     [on]  16:00 - 19:00
  Friday       [on]  16:00 - 21:00
  Saturday     [on]  10:00 - 21:00
  Sunday       [on]  10:00 - 18:00
```

## Daily limits

- [ ] Video has an optional persisted daily quota in minutes.
- [ ] Gaming has an optional persisted daily quota in minutes.
- [ ] A disabled quota means unlimited daily use, while schedule restrictions may still apply.
- [ ] The Web Admin must show the configured limit, time already used today, and remaining time.
- [ ] The owner can change the quota without SSH or editing files.
- [ ] Video and Gaming usage counters are independent.
- [ ] Usage resets at local midnight according to the configured device timezone.
- [ ] Changing a configured quota must not silently erase already consumed time for the current day.
- [ ] Resetting today's usage is a separate explicit admin action with confirmation; simply changing settings must not reset usage.

## Weekly schedules

- [ ] Video and Gaming each have an independent weekly schedule.
- [ ] Each weekday can be enabled or disabled independently.
- [ ] Each enabled weekday supports at least one allowed `from` / `to` time range.
- [ ] Prefer supporting multiple allowed windows per day so configurations such as `07:00-08:00` and `16:00-19:00` do not require awkward workarounds.
- [ ] A weekday with no configured window means the mode is unavailable that day when schedule enforcement is enabled.
- [ ] The Web Admin must validate that time windows are syntactically valid and do not accidentally overlap in contradictory ways.
- [ ] Overnight windows such as `20:00-01:00` must either be explicitly supported with clear semantics or rejected with a clear validation message; do not interpret them ambiguously.
- [ ] Schedule changes should take effect without reboot where practical.

## Time accounting

Usage time must be measured at the box level because third-party providers do not expose consistent playback/game-state APIs.

Default rule:

```text
Video usage counts while VideoMode owns the foreground.
Gaming usage counts while GamingMode owns the foreground.
```

Therefore:

- opening the Video/Gaming launcher does not consume quota until the corresponding foreground provider session starts;
- time continues counting while a provider is paused, in a provider menu, or waiting on a cloud session, because those states are not reliably observable across all services;
- returning to Mello stops the active category timer;
- switching from one Video provider to another during the same foreground Video session continues the same Video quota;
- switching from Steam Link to Shadow continues the same Gaming quota.

Do not depend on Netflix, Steam, Shadow, GeForce NOW, or another provider reporting trustworthy watch/play state for enforcement.

## Persistence and reboot behavior

The system must not make daily limits easy to bypass through restart or power cycling.

- [ ] Persist consumed Video/Gaming seconds regularly while a limited mode is active.
- [ ] Also persist on clean mode exit, shutdown, restart, and before update where practical.
- [ ] After reboot, restore today's usage from persisted state.
- [ ] Store usage by local calendar date plus category rather than one lifetime counter.
- [ ] Retain only the history needed for operation/admin visibility; do not build unnecessary behavioral analytics into this feature.
- [ ] Handle an unclean power loss conservatively enough that only a small amount of recent usage can be lost.

## Clock, timezone, and DST

Time rules depend on a trustworthy clock.

- [ ] Use the Raspberry Pi's configured local timezone for weekday, schedule, and daily-reset calculations.
- [ ] Show the active timezone and current device time in the Web Admin near schedule settings.
- [ ] Prefer network time synchronization when available.
- [ ] Do not silently treat an obviously invalid system date/time as a valid unrestricted schedule state.
- [ ] Define deterministic behavior for daylight-saving-time transitions so repeated/skipped local times do not reset quota or grant an unintended extra daily allowance.
- [ ] A timezone change must not automatically reset today's consumed quota.

## Launch behavior when blocked

A blocked mode/provider must fail clearly rather than appear broken.

Examples of user-facing reasons:

```text
Gaming is available today from 16:00 to 19:00.

Gaming time for today has been used up.
Available again tomorrow.

Video is not available on Sundays.

Video has 12 minutes remaining today.
```

- [ ] The normal UI may still show the Video/Gaming category when it is temporarily time-blocked, but it must communicate the reason and remaining/next available time clearly.
- [ ] Do not confuse `temporarily blocked by policy` with `disabled by owner` or `unavailable on this hardware`.
- [ ] Web Admin status must expose the current policy result and reason.

## Active-session enforcement

Time limits are enforced during a running session, not only at launch.

- [ ] Continuously evaluate the current quota and active schedule boundary.
- [ ] Warn the user before forced exit, with sensible milestones such as 15, 5, and 1 minute remaining where applicable.
- [ ] If the daily quota reaches zero, close the active Video/Gaming provider cleanly and return to Mello.
- [ ] If the configured allowed time window ends, close the active provider cleanly and return to Mello even when daily quota remains.
- [ ] Save usage before ending the provider.
- [ ] Do not abruptly power off the Raspberry Pi as a time-limit mechanism.
- [ ] Recovery after a provider ignores/blocks a close request must use the existing ModeManager/watchdog path and still return to Mello.

## Parent/admin override

A future implementation may offer a deliberate temporary parent override, but it must not be implicit.

If implemented:

- require the Web Admin (and its password/session when password protection is enabled);
- make the override explicit and time-bounded, for example `+15 minutes`, `+30 minutes`, or `allow until schedule end`;
- log only the control action and duration, not provider account/session data;
- do not permanently change the weekly schedule or normal daily limit unless the owner chooses to edit those settings.

A permanent `ignore limits` child-facing button is not allowed.

## Suggested settings model

```text
usage_limits:
  timezone: Europe/Berlin

  video:
    enabled: true
    daily_limit_minutes: 60 | null
    schedule_enabled: true
    weekly_windows:
      monday:
        - 15:00-19:00
      tuesday:
        - 15:00-19:00
      wednesday:
        - 15:00-19:00
      thursday:
        - 15:00-19:00
      friday:
        - 15:00-20:00
      saturday:
        - 09:00-20:00
      sunday: []

  gaming:
    enabled: true
    daily_limit_minutes: 90 | null
    schedule_enabled: true
    weekly_windows:
      monday:
        - 16:00-19:00
      ...
```

Consumed usage should be persisted separately from policy configuration so changing a schedule does not accidentally rewrite today's accounting.

## Security

- Time-limit/schedule changes are privileged Web Admin configuration mutations.
- When optional Admin password protection is enabled, changing limits, schedules, timezone, today's usage, or temporary overrides requires authentication.
- Apply the same CSRF and request-validation rules as other Web Admin mutations.
- Never derive a shell command from submitted time/schedule values.
- Server-side validation is authoritative; browser-side validation is only for user experience.

## Testing

Add tests for at least:

- independent Video and Gaming daily quotas;
- unlimited quota + schedule-only operation;
- quota-only operation with schedule disabled;
- weekday allow/deny behavior;
- multiple windows on one day;
- launch immediately before/after a boundary;
- quota exhaustion during an active session;
- schedule end during an active session;
- quota persistence across reboot/update;
- local-midnight reset;
- timezone changes and DST transitions;
- invalid clock behavior;
- remote Admin policy change while Video/Gaming is active;
- blocked reason exposed consistently to touchscreen and Web Admin;
- service disabled vs unavailable vs time-blocked states.

## Exit criteria

Usage controls are complete when an owner can independently configure Video and Gaming so that:

1. each category can have its own daily usage limit;
2. each category can have allowed hours per weekday;
3. the box shows remaining time and why a mode is currently blocked;
4. usage cannot be trivially reset by rebooting the device;
5. active sessions end safely when their quota or allowed window expires;
6. settings and accounting survive normal reboot/update;
7. the same policy is enforced through touchscreen, Web Admin/API, native Gaming clients, and browser kiosk providers.
