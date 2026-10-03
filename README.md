# ZEN32 / ZEN35 Scene & LED Control - LTek

LED state tracking and scene control for the **Zooz ZEN32** and **ZEN35** scene
controllers over Z-Wave JS. Each button's LED can mirror the state of any entity
(light, cover, lock, climate, sensor, etc.) while the button presses run your
own scenes/actions.

Based on `ZEN32-control-track.yaml` by [rwalker777](https://github.com/rwalker777/Zen32-HA-Blueprint)
(original v2026.9.5); see the [Credits](#credits) section. This is a standalone
project, not a GitHub fork.

**Requires Home Assistant 2024.10.0 or newer.**

## Install

[![Open your Home Assistant instance and show the blueprint import dialog with this blueprint pre-filled.](https://my.home-assistant.io/badges/blueprint_import.svg)](https://my.home-assistant.io/redirect/blueprint_import/?blueprint_url=https%3A%2F%2Fraw.githubusercontent.com%2FLTek%2Fzen32-zen35-control-ltek%2Fmain%2Fzen32-zen35-control-ltek.yaml)

Click the button above, or in Home Assistant go to **Settings → Automations &
Scenes → Blueprints → Import Blueprint** and paste:

```
https://raw.githubusercontent.com/LTek/zen32-zen35-control-ltek/main/zen32-zen35-control-ltek.yaml
```

This installs the blueprint under `blueprints/automation/LTek/`, so your
automations reference it as:

```yaml
use_blueprint:
  path: LTek/zen32-zen35-control-ltek.yaml
```

> The import dialog shows the install path in an editable field, so you can
> adjust it there if you ever want a different folder.

Example automation: [`examples/living-room-zen35.yaml`](examples/living-room-zen35.yaml)

## Features

- **Per-button LED state tracking.** Point a button at an entity; its LED color,
  behavior and brightness reflect that entity's on / off / unavailable state.
- **Global Defaults.** One top-level section sets color, behavior and brightness
  for on / off / unavailable states across **all** buttons. Any button that sets
  its own value for an item overrides the matching default — otherwise it inherits
  from Global Defaults.
- **Actions per button.** Pressed once, held, released, and 2x–5x multi-tap,
  each with its own action sequence. Includes the big relay button and a 3-way
  switch (momentary, 800-series or 700-series firmware 10.40+).
- **Brightness on the device scale.** LED brightness is chosen on the device's own
  1–10 scale (3 = Low, 6 = Medium, 10 = Bright). New-scale firmware (ZEN35 1.40+,
  ZEN32 800LR 3.30+) is auto-detected; older firmware is remapped to its 3 levels
  automatically. Detection can be manually overridden.
- **Conditional Override.** Supply your own triggers and condition — any time,
  entity, sun, numeric or template logic Home Assistant supports — then set
  override behavior, color and brightness. Values are **per button**, so each LED
  can react differently (or not at all) to the same condition. Replaces the old
  fixed Night Time feature.
- **Relay / dimmer options.** Smart-bulb mode and load/LED reporting (parameters
  19 & 20), plus optional "turn off untracked LEDs." The big button's LED can be
  driven by this blueprint or left to the device firmware.

## How it works

The automation listens for Z-Wave scene notifications (button presses → Actions)
and for state changes on the tracked entities (→ LED updates). On each tracked
state change it writes three Z-Wave config parameters for that button —
behavior, color, and brightness — with a short delay between writes. A
startup/reload refresh re-syncs every LED to current entity state.

Each button's settings live in two panels: **Actions** (what the presses do) and
**LED Settings** (the entity to track and how the LED looks).

### Resolution order

Each of the three LED values — behavior, color, brightness — is resolved
independently, highest priority first:

| Priority | Source |
|---|---|
| 1 | **Conditional Override** value, while the override condition is true |
| 2 | **Per-button** value in that button's LED Settings panel |
| 3 | **Global Defaults** |

An override field set to **No Change** falls through to the normal per-state
value, so you can dim every LED at night while each keeps its own colors.

### Conditional Override

Two inputs define *when*:

- **Override Triggers** — when to re-check (e.g. a time trigger at 22:00 and
  another at 07:00). These only wake the automation.
- **Override Condition** — whether the override is active (e.g. a time condition
  for after 22:00). This is checked on *every* LED write, so the LEDs are also
  correct after a restart, when no trigger fired.

Both are usually the same idea expressed twice. That is deliberate: a trigger is
an edge and a condition is a level, and the LEDs need the level to come up
correctly at startup.

Then set the values — globally in the same panel, or per button in that button's
LED Settings panel. A button whose override values are all **No Change** ignores
the override entirely and keeps tracking normally.

```yaml
# Dim every LED to 2 between sunset and sunrise, keeping each button's colors.
override_triggers:
  - trigger: sun
    event: sunset
  - trigger: sun
    event: sunrise
override_condition:
  - condition: sun
    after: sunset
    before: sunrise
global_ovr_brightness: '2'
```

`after: sunset` + `before: sunrise` is Home Assistant's canonical "when dark"
test — it holds from sunset until midnight and from midnight until sunrise. For a
fixed clock schedule, swap in `trigger: time` / `at:` and `condition: time` /
`after:` / `before:`. Inside the polar circles, use a sun-elevation template
(`state_attr('sun.sun','elevation') < -6`) instead.

> **Leave the Override Condition empty and the override runs all the time.** An
> empty condition list is vacuously true in Home Assistant. That is harmless
> while the override values are all **No Change**, but once you set one — a
> brightness, say — it applies 24/7 until you add the condition.

### The big button (relay / dimmer)

The big button drives a real load — a **relay** on the ZEN32, a **dimmer** on the
ZEN35 — so its LED can come from either of two places. **LED Control Mode** picks
which:

| Mode | LED follows | Color / brightness | Night override | Z-Wave traffic |
|---|---|---|---|---|
| **LED uses Blueprint setting** (default) | your Entity to Track | yes | yes | a write per change |
| **LED uses Buttons Firmware State** | the attached load, in firmware | **no** — on/off only | no | none |

The two are mutually exclusive. In firmware mode the Entity to Track and every LED
value on that button — including the override ones — are ignored, and
**Firmware LED State** chooses whether the LED is on when the load is on or when
it's off.

Blueprint mode is the default because it makes the big button behave like the four
scene buttons. Firmware mode is still the better pick if the big button only
switches its own load and you want the LED instant and silent on the mesh.

## Setup

1. Select your ZEN32 / ZEN35 device.
2. (Optional) Set **Global Defaults** for the common look, then only customize the
   buttons that differ.
3. For each button, open **LED Settings**, pick the entity to track and, if
   desired, override color / behavior / brightness (leave as **Global Default**
   to inherit).
4. Open **Actions** and add the presses you want to use.
5. Configure the Conditional Override and relay options if needed.

> **Note:** per-button color/behavior/brightness default to **Global Default**.
> When upgrading an existing automation to this version, any option left at
> default will follow Global Defaults; explicitly chosen values are preserved.

## Upgrading from a version with Night Time

Night Time settings are **not** migrated — the old inputs no longer exist, so
their values are dropped silently. Reconfigure as follows:

| Old Night Time input | Replacement |
|---|---|
| `schedule_start` / `schedule_stop` | Two **Override Triggers** (time, at each) plus an **Override Condition** (time, after/before) |
| `night_entity` | One **Override Trigger** (state, on that entity) plus an **Override Condition** (state, that entity = `on`) |
| `invert_night_entity` | Invert the condition itself (`not` → state is `off`) |
| `night_led_brightness` (Low/Medium/Bright) | **Override Brightness** on the 1–10 scale (3 / 6 / 10) — globally, or per button |
| `night_led_color` | **Override Color** — globally, or per button |
| Behavior set to `On during daytime` | Behavior `Always On` + **Override Behavior** `Always Off` |

Two improvements come with the move: override brightness is now set per button
rather than one value flattening every LED, and it is scaled for the detected
firmware (the old night path accepted only the three names, so a numeric value
silently became "no change").

---

# Changelog

Current version: **v2026.10.03.2**. See [`CHANGELOG.md`](CHANGELOG.md)
for the full detailed history.

- **2026.10.03.2** — **LED Control Mode** on the big button: blueprint-driven
  (new default) or firmware-driven. "Relay" wording is now "relay / dimmer,"
  since the ZEN35's big button is a dimmer.
- **2026.10.03.1** — **Night Time replaced by Conditional Override** (user-supplied
  triggers + condition, per-button override values). Button panels split into
  **Actions** and **LED Settings**. Requires HA 2024.10+. **Breaking:** night
  settings are not migrated.
- **2026.10.02.2** — UI usability changes (layout, icons, labels).
- **2026.10.02.1** — Added **Global Defaults**; button number icons + cleaned-up
  labels; "Multipress" renamed to "Scene Actions."
- **2026.10.01.2** — 1–10 brightness scale with automatic legacy remapping.
- **From original (v2026.9.5)** — Brightness values always validated before writing.

---

# Credits

Based on the **ZEN32 and ZEN35 Scene and State** blueprint by
[rwalker777](https://github.com/rwalker777/Zen32-HA-Blueprint) (original
v2026.9.5), which remains the upstream source. This project is an independently
maintained rework — not a GitHub fork — and tracks its own version history in
[`CHANGELOG.md`](CHANGELOG.md). Two bug fixes in v2026.10.03.2 were identified by
cross-referencing the [angelo-labs](https://github.com/angelo-labs/Zen32-HA-Blueprint)
fork of the same upstream.
