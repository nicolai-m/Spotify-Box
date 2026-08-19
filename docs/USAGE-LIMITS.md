# Video and Gaming Usage Limits

This document defines optional parental time controls for the Video and Gaming layers. The controls are configured from the local Web Admin and enforced by the shared settings/device-control layer, not by individual provider pages.

The owner may use a daily budget, a weekly schedule, both together, or neither. Video and Gaming are configured independently.

## Scope

Video and Gaming each have two independent time-control switches:

```text
daily_limit_enabled
schedule_enabled
```

Both switches are optional and independent.

Valid configurations include:

```text
Daily budget OFF + Weekly schedule OFF
  -> no time restriction

Daily budget ON + Weekly schedule OFF
  -> only the daily quota is enforced

Daily budget OFF + Weekly schedule ON
  -> only weekday/time windows are enforced

Daily budget ON + Weekly schedule ON
  -> both rules are enforced; the strictest current limit wins
```

Audio is not time-limited by this feature unless a separate roadmap item explicitly adds Audio limits later.

## Example policies

```text
Video
  daily limit: enabled, 60 minutes
  weekly schedule: disabled

Gaming
  daily limit: disabled
  weekly schedule: enabled
    Monday-Friday: 16:00-19:00
    Saturday: 10:00-21:00
    Sunday: 10:00-18:00
```

Another device may use neither:

```text
Video
  daily limit: disabled
  weekly schedule: disabled

Gaming
  daily limit: disabled
  weekly schedule: disabled
```

In that configuration Video/Gaming are unrestricted by time policy and are controlled only by the normal global mode, provider enablement, and hardware availability rules.

## Core permission rule

A Video or Gaming provider may launch only when all **enabled** relevant rules pass:

```text
mode globally enabled
AND provider individually enabled
AND provider available on this hardware
AND (schedule disabled OR current time is inside an allowed window)
AND (daily limit disabled OR daily quota has remaining time)
```

Disabled time-policy components must not accidentally restrict usage.

The strictest enabled rule always wins.

Examples:

- Gaming daily quota is enabled with 45 minutes remaining, but its enabled schedule window ends in 10 minutes -> only 10 minutes are currently available.
- Video daily quota is disabled, but its weekly schedule is enabled for 15:00-18:00 -> Video cannot launch outside that window.
- Gaming weekly schedule is disabled, but its 90-minute daily quota is enabled -> Gaming is available at any time until the quota is exhausted.
- Both Video time controls are disabled -> Video has no time restriction.
- Steam Link is enabled, but `gaming_enabled = false` -> it still cannot launch.

## Web Admin UI

Add a dedicated `Time limits` / `Usage limits` section, or equivalent controls inside the Video and Gaming settings pages.

Suggested structure:

```text
Time limits

VIDEO
  Daily budget                  [on]
  Daily limit                   60 min
  Used today                    23 min
  Remaining today               37 min

  Weekly schedule               [off]
  Monday                        15:00 - 19:00
  Tuesday                       15:00 - 19:00
  ...

GAMING
  Daily budget                  [off]
  Daily limit                   90 min

  Weekly schedule               [on]
  Monday                        16:00 - 19:00
  Tuesday                       16:00 - 19:00
  Wednesday                     16:00 - 19:00
  Thursday                      16:00 - 19:00
  Friday                        16:00 - 21:00
  Saturday                      10:00 - 21:00
  Sunday                        10:00 - 18:00
```

When a control is disabled, its configured values may remain visible/editable but must clearly be marked as inactive. This allows a parent to temporarily turn a rule off without losing the setup.

## Daily budgets

- [ ] Video has an independent persisted `daily_limit_enabled` switch.
- [ ] Gaming has an independent persisted `daily_limit_enabled` switch.
- [ ] Video and Gaming each have an optional configured daily quota in minutes.
- [ ] Turning the daily budget off means unlimited daily use from the quota perspective; an enabled weekly schedule may still restrict access.
- [ ] Turning the daily budget off must preserve the configured minute value so it can be re-enabled later without re-entering it.
- [ ] While the daily budget is disabled, usage does not count against that daily quota.
- [ ] If the daily budget is re-enabled later the same day, only usage accumulated while the budget was enabled counts unless an explicit future policy changes this behavior.
- [ ] The Web Admin must show the configured limit plus used/remaining time only when useful, and clearly indicate when the quota is disabled.
- [ ] The owner can enable/disable or change the quota without SSH or editing files.
- [ ] Video and Gaming usage counters are independent.
- [ ] Active daily usage resets at local midnight according to the configured device timezone.
- [ ] Changing a configured quota must not silently erase already consumed time for the current day.
- [ ] Turning the daily-budget switch off/on must not silently reset already recorded quota usage for that day.
- [ ] Resetting today's usage is a separate explicit admin action with confirmation.

## Weekly schedules

- [ ] Video has an independent persisted `schedule_enabled` switch.
- [ ] Gaming has an independent persisted `schedule_enabled` switch.
- [ ] Turning the weekly schedule off means all weekdays/times are allowed from the schedule perspective; an enabled daily budget may still restrict access.
- [ ] Turning the schedule off must preserve all configured weekday/time windows for later re-enabling.
- [ ] Each weekday can be enabled or disabled independently while schedule enforcement is enabled.
- [ ] Each enabled weekday supports at least one allowed `from` / `to` time range.
- [ ] Prefer supporting multiple allowed windows per day so configurations such as `07:00-08:00` and `16:00-19:00` do not require awkward workarounds.
- [ ] A weekday with no configured window means the mode is unavailable that day only when schedule enforcement is enabled.
- [ ] The Web Admin must validate time windows server-side and avoid contradictory/overlapping configuration.
- [ ] Overnight windows such as `20:00-01:00` must either be explicitly supported with clear semantics or rejected with a clear validation message.
- [ ] Schedule changes and enabling/disabling schedule enforcement should take effect without reboot where practical.

## Time accounting

Usage time must be measured at the box level because third-party providers do not expose consistent playback/game-state APIs.

Default rule while the corresponding daily budget is enabled:

```text
Video usage counts while VideoMode owns the foreground.
Gaming usage counts while GamingMode owns the foreground.
```

Therefore:

- opening the Video/Gaming launcher does not consume quota until the corresponding foreground provider session starts;
- time continues counting while a provider is paused, in a provider menu, or waiting on a cloud session, because those states are not reliably observable across all services;
- returning to Mello stops the active category timer;
- switching between Video providers continues the same Video quota;
- switching between Gaming providers continues the same Gaming quota;
- when the relevant daily-budget switch is disabled, foreground time is not deducted from that daily quota.

Do not depend on Netflix, Steam, Shadow, GeForce NOW, or another provider reporting trustworthy watch/play state for enforcement.

## Persistence and reboot behavior

The system must not make an enabled daily limit easy to bypass through restart or power cycling.

- [ ] Persist consumed Video/Gaming seconds regularly while the corresponding daily budget is enabled and a counted mode is active.
- [ ] Also persist on clean mode exit, shutdown, restart, and before update where practical.
- [ ] After reboot, restore today's recorded usage.
- [ ] Store usage by local calendar date plus category rather than one lifetime counter.
- [ ] Preserve usage data when a daily budget is temporarily switched off so toggling it is not a reset mechanism.
- [ ] Retain only the history needed for operation/admin visibility; do not build unnecessary behavioral analytics into this feature.
- [ ] Handle an unclean power loss conservatively enough that only a small amount of recent counted usage can be lost.

## Clock, timezone, and DST

Time rules depend on a trustworthy clock whenever at least one schedule or daily-budget policy uses calendar-day boundaries.

- [ ] Use the Raspberry Pi's configured local timezone for weekday, schedule, and daily-reset calculations.
- [ ] Show the active timezone and current device time in the Web Admin near schedule settings.
- [ ] Prefer network time synchronization when available.
- [ ] Do not silently treat an obviously invalid system date/time as a valid unrestricted schedule state when a time rule is enabled.
- [ ] If all Video/Gaming time controls are disabled, an invalid clock must not by itself block those modes.
- [ ] Define deterministic behavior for daylight-saving-time transitions so repeated/skipped local times do not reset quota or grant an unintended extra allowance.
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
- [ ] When both time controls for a category are disabled, do not show a misleading time-blocked state.

## Active-session enforcement

Only enabled time controls are enforced during a running session.

- [ ] Continuously evaluate any enabled quota and schedule boundary.
- [ ] Warn the user before forced exit, with sensible milestones such as 15, 5, and 1 minute remaining where applicable.
- [ ] If an enabled daily quota reaches zero, close the active Video/Gaming provider cleanly and return to Mello.
- [ ] If an enabled schedule's current allowed window ends, close the active provider cleanly and return to Mello even when daily quota remains.
- [ ] If the parent disables the daily quota or schedule while a session is running, stop enforcing that specific rule immediately without ending an otherwise permitted session.
- [ ] If the parent enables a stricter rule while a session is running and the session is already outside that rule, apply the new policy consistently and return to Mello after clear feedback.
- [ ] Save counted usage before ending a provider.
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
    daily_limit_enabled: true
    daily_limit_minutes: 60
    schedule_enabled: false
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
    daily_limit_enabled: false
    daily_limit_minutes: 90
    schedule_enabled: true
    weekly_windows:
      monday:
        - 16:00-19:00
      ...
```

Configured values are preserved while their respective switch is off. Consumed usage is persisted separately from policy configuration so changing or toggling a schedule does not accidentally rewrite today's accounting.

## Security

- Time-limit/schedule changes are privileged Web Admin configuration mutations.
- When optional Admin password protection is enabled, enabling/disabling or changing limits, schedules, timezone, today's usage, or temporary overrides requires authentication.
- Apply the same CSRF and request-validation rules as other Web Admin mutations.
- Never derive a shell command from submitted time/schedule values.
- Server-side validation is authoritative; browser-side validation is only for user experience.

## Testing

Add tests for at least:

- both daily budget and weekly schedule disabled;
- daily-budget-only operation;
- schedule-only operation;
- both rules enabled together;
- toggling either rule without losing its configuration;
- toggling the daily budget without resetting recorded usage;
- independent Video and Gaming policies;
- weekday allow/deny behavior;
- multiple windows on one day;
- launch immediately before/after a boundary;
- quota exhaustion during an active session;
- schedule end during an active session;
- disabling/enabling a rule during an active session;
- quota persistence across reboot/update;
- local-midnight reset;
- timezone changes and DST transitions;
- invalid clock behavior with rules enabled and all rules disabled;
- remote Admin policy change while Video/Gaming is active;
- blocked reason exposed consistently to touchscreen and Web Admin;
- service disabled vs unavailable vs time-blocked states.

## Exit criteria

Usage controls are complete when an owner can independently configure Video and Gaming so that:

1. the daily budget can be enabled or completely disabled;
2. the weekly schedule can be enabled or completely disabled;
3. either control can be used alone, both can be combined, or both can be off;
4. disabled controls preserve their configuration for later re-enabling;
5. the box shows remaining time and why a mode is currently blocked when a rule is active;
6. enabled daily usage cannot be trivially reset by rebooting or toggling the control;
7. active sessions end safely when an enabled quota or allowed window expires;
8. settings and accounting survive normal reboot/update;
9. the same policy is enforced through touchscreen, Web Admin/API, native Gaming clients, and browser kiosk providers.
