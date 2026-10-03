# Changelog — ZEN32 / ZEN35 Scene & LED Control - LTek

Blueprint file: `zen32-zen35-control-ltek.yaml`. Based on
`ZEN32-control-track.yaml` by rwalker777 (original v2026.9.5); independently
maintained, not a GitHub fork. `source_url` is intentionally omitted so Home
Assistant will not re-import over this copy.

Filenames in historical entries below (`ZEN32-control-track.yaml`,
`living-room-zen35-v2026.10.03.1.yaml`) reflect the names in use at the time.

---

## 2026.10.03.2 (current)

The big button's LED is now blueprint-driven by default, and "relay" wording is
corrected for the ZEN35.

### Fixed
Two bugs confirmed in the shared rwalker777 lineage (found by reviewing the
angelo-labs fork) and present in this fork too:
- **Held and Released scene actions fired for the wrong event.** The
  `value_to_name` map had Central Scene key `1` as `KeyHeldDown` and `2` as
  `KeyReleased`. Z-Wave JS defines the reverse (`CentralSceneKeys.KeyReleased =
  0x01`, `KeyHeldDown = 0x02`, verified against zwave-js source), and the
  downstream action branches match on the name, so a physical hold ran the
  Released action and vice versa. Swapped to `1: KeyReleased`, `2: KeyHeldDown`.
- **A climate entity in cooling mode never counted as "on."** `on_states` listed
  `cold`, which is not a Home Assistant HVAC mode; the correct string is `cool`.
  A thermostat cooling therefore had its LED treated as off. Corrected to `cool`.

Not adopted from that fork: its manual "LED Brightness Precision" (legacy/fine)
toggle — this fork already auto-detects the firmware brightness scale and remaps
legacy devices, which is strictly more capable — and its null-value LED guard and
button-4 label fix, both of which this fork already carries. Its remaining
commits are nighttime-feature changes, superseded here by the Conditional
Override.

### Added
- **`led_mode0` — "LED Control Mode"** on the big button, first input in its LED
  Settings panel:
  - `LED uses Blueprint setting` (`blueprint`) — **new default.** The blueprint
    drives the LED from Entity to Track, with per-state colors, brightness and the
    Conditional Override, exactly like the four scene buttons.
  - `LED uses Buttons Firmware State` (`firmware`) — the device firmware drives the
    LED from its own attached load. On/off only; no color, brightness or override.
- `led_mode` added to each entry of the `inputs[]` trigger variable
  (`blueprint` for buttons 1-4, which are never firmware-driven).

### Changed
- **Default behavior for the big button flips.** `track_relay0` previously
  defaulted to `'0'` (firmware, on-when-load-off), which silently disabled the
  entity tracking and every LED value on that button. A fresh automation now gets
  a big button that behaves like the others. **Existing automations that relied on
  firmware tracking must set `led_mode0: firmware`** — see Compatibility.
- **`track_relay0` is now "Firmware LED State"** and only applies in firmware
  mode. Its `Disabled` (`'99'`) option is gone — that job belongs to `led_mode0`
  now — leaving just the two real firmware choices (`'0'`, `'1'`), default `'1'`.
- **Mode checks switched from `track_relay` to `led_mode`.** `valid_buttons`,
  `untracked_buttons` and the per-button state-trigger gate now test
  `led_mode == 'firmware'` instead of `track_relay in ['0', '1']`. The old test
  conflated "which way does the firmware track the load" with "who owns the LED,"
  which is why `'0'`/`'1'` were overloaded as an off switch for the whole button.
- **Parameter 1 write on reload** is now
  `{{ inputs[0].track_relay if inputs[0].led_mode == 'firmware' else '2' }}` — the
  firmware value in firmware mode, otherwise `'2'` (Always off) to hand the LED to
  the blueprint. Previously `'2' if track_relay == '99' else track_relay`.
- **"Relay" renamed to "Relay / Dimmer"** in user-facing strings: both big-button
  panel titles, and the parameter 19 / 20 names and descriptions. The ZEN32's big
  button switches a relay; the **ZEN35's is a dimmer** (Z-Wave JS labels the device
  "Dimmer & Scene Controller" and parameter 1 "LED Indicator (Dimmer)"), so the
  single term was wrong on half the supported hardware. Input keys are unchanged.

### Compatibility
- **Breaking for firmware-tracked big buttons.** `track_relay0: '0'` or `'1'` no
  longer suppresses blueprint control on its own — `led_mode0` does. An existing
  automation with `track_relay0` set and no `led_mode0` picks up the new
  `blueprint` default, so the big button will start tracking its entity and
  writing colors. Add `led_mode0: firmware` to keep the old behavior.
- `track_relay0: '99'` from an older config is no longer a valid option. It meant
  "blueprint controls this LED," which is the new default, so dropping the key
  reproduces the old behavior. A leftover `'99'` is inert in blueprint mode (the
  value is only read in firmware mode).
- Buttons 1-4 are unaffected.

### Example config
`living-room-zen35-v2026.10.03.1.yaml` updated: `track_relay0: '1'` replaced by
`led_mode0: blueprint`, which activates the previously inert `input_entity0`,
`on_color0` and `unavail_color0`. All five buttons now dim after sunset.

### Testing
- Blueprint re-parsed; 128 declared inputs and 128 `!input` references, no orphans
  either direction.
- `led_mode0` default confirmed `blueprint`; `track_relay0` default `'1'` confirmed
  to be one of its own options (the earlier `'99'` default would have been
  unselectable after the option was removed).
- Simulated `valid_buttons`, `untracked_buttons` and the parameter 1 write across
  6 cases — both modes x entity present/absent, plus a stale firmware value in
  blueprint mode. Blueprint mode writes button 0 and sends param 1 = `'2'`;
  firmware mode skips the button and sends the chosen firmware value; blueprint
  mode with no entity routes button 0 to the forced-off list.
- Example config re-rendered through the blueprint's resolution templates: 31 keys
  all declared, all select values legal, 5 buttons x 3 states correct by day and
  dimmed to 2 after dark.
- Not verified on hardware.

---

## 2026.10.03.1

The Night Time feature is replaced by a general **Conditional Override**, and the
per-button panels are split by function. **Requires Home Assistant 2024.10.0 or
newer** (`min_version` raised from 2024.6.0).

### Added
- **Conditional Override section** (`mdi:tune-variant`, collapsed) with 5 inputs:
  - **Override Triggers** (`selector: trigger:`) — user-supplied triggers that
    cause the LEDs to be re-evaluated. Merged into the blueprint's own trigger
    list using the `- triggers: !input …` syntax.
  - **Override Condition** (`selector: condition:`) — evaluated on *every* LED
    write to decide whether the override applies.
  - **Override Behavior / Color / Brightness** — the global override values,
    each defaulting to `No Change` (`'99'`).
- **15 per-button override inputs** (`ovr_state0-4`, `ovr_color0-4`,
  `ovr_brightness0-4`), each defaulting to `global`. A button can set its own
  override value, inherit the global one, or opt out with `No Change`.
- `map_b_ovr` brightness resolver in the shared variables block — same
  name / 1-10 / legacy-remap logic as `map_b_off`, so override brightness is
  scaled for the detected firmware. The old night path used a names-only map in
  which a numeric value silently resolved to `99` (no change).
- Variables `g_ovr_state`, `g_ovr_color`, `g_ovr_brightness`; `inputs[]` gained
  `ovr_state`, `ovr_color`, `ovr_brightness` per button.

### Changed
- **Button panels split by function.** Each button now has an **Actions** panel
  (Pressed Once, Held, Released, 2x-5x) and a **LED Settings** panel (LED Track
  Relay where applicable, Entity to Track, the nine per-state values, and the
  three override values). `Pressed Once` moved from the old settings panel into
  Actions; `Entity to Track` stayed with the LED values because it feeds the LED
  state machine, not the button presses. Section keys are now
  `buttonN_actions` / `buttonN_led`; the former `buttonN_multipress` sections are
  gone. Input keys are unchanged — section membership is presentational only.
- **3-way panel** renamed to `3 Way Switch - Actions`; its momentary / firmware
  requirements moved from the title into the description.
- **Write block is now a two-branch `choose`** — an override branch and a default
  branch — because a `condition:` input is a config structure and cannot be
  evaluated inside a Jinja template.
- **Refresh gate inverted** to
  `{{ trigger.id not in ['ButtonPress', 'StateTrigger', 'AutomationUpdate'] }}`,
  so the refresh runs for Home Assistant start *and* for any user-supplied
  override trigger, whose IDs cannot be known in advance.
- **Top-level `trigger:` is now `triggers:`**, required for the nested merge.
  Individual triggers keep the singular `platform:` style, which remains valid.
- `global_off_state` default `'99'` -> `'3'`. No behavioral change: `'99'`
  already wrote `'3'` whenever night was unconfigured.
- `LED Track Relay` description no longer references nighttime.

### Removed
- The **Night Time section** and all 6 inputs: `night_entity`,
  `invert_night_entity`, `schedule_start`, `schedule_stop`,
  `night_led_brightness`, `night_led_color`.
- Variables `in_nighttime`, `night_states`, `night_led_brightness`,
  `night_led_color`, `night_entity`, `invert_night_entity`, `brightness_map`,
  `brightness_map_new`, `brightness_map_legacy`; trigger variables
  `schedule_start`, `schedule_stop`, `schedule_set`.
- Triggers `NightTime`, `DayTime`, `EntityToggle`.
- **The `On during daytime` (`'99'`) option, from all 18 behavior dropdowns.** It
  was a hidden conditional in a behavior costume — it meant "Always On, except
  Always Off while night is active." That is now stated directly as
  `Behavior = Always On` plus `Override Behavior = Always Off`, which responds to
  any condition rather than only the night schedule. `'99'` now has exactly one
  meaning (No Change) plus the unrelated `LED Track Relay` "Disabled".

### Documentation
- README rewritten around the new model: resolution-order table, a Conditional
  Override section with a worked YAML example, a note on relay tracking
  suppressing the big button's LED writes, and an **Upgrading from a version with
  Night Time** table mapping each retired input to its replacement.
- `living-room-zen35-v2026.10.03.1.yaml` added as a migrated example config
  (see Example config below). The pre-migration `living-room-zen35.yaml` is kept
  for comparison.

### Compatibility
- **Breaking: Night Time settings are not migrated.** The old inputs no longer
  exist, so their values are dropped silently. An automation that used night time
  must be reconfigured with Override Triggers and an Override Condition. The
  equivalent of the old schedule is two time triggers plus a time condition — see
  the README's upgrade table, or
  `living-room-zen35-v2026.10.03.1.yaml` for a worked example.
- A leftover `'99'` behavior value from an older config is coerced to `'3'`
  (Always On) before the write, so it cannot reach the device as an invalid
  parameter value. This matches what `'99'` already did without night configured.
- Per-button override values default to `global`, and the global override values
  default to `No Change`, so an upgraded automation that ignores the new section
  behaves exactly as it did before.

### Example config
`living-room-zen35-v2026.10.03.1.yaml` is the Living Room ZEN35 automation on this
version, with the override keyed to **sunset / sunrise**:
- **Override condition supplied.** It had been left empty (`[]`), which is
  vacuously true, so `global_ovr_brightness: '2'` was dimming every tracked LED
  around the clock rather than only after dark.
- **Override triggers rewritten to valid trigger YAML.** The UI editor had
  serialized them as `trigger: sun.sunset` with an `options:` block holding an
  all-zero offset and `offset_type: before`. Home Assistant's sun trigger is
  `trigger: sun` + `event: sunset`; the zero offset was a no-op, so nothing was
  lost in the conversion.
- Override is now two `sun` triggers (sunset, sunrise) plus
  `condition: sun / after: sunset / before: sunrise` — Home Assistant's canonical
  "when dark" test. All per-button `ovr_*` inputs stay at `global`, so every
  tracked LED dims to 2 after dark and keeps its own per-state color.
- Flagged, not changed: `input_entity0` / `on_color0` / `unavail_color0` do
  nothing while `track_relay0` is `'1'`, because both `valid_buttons` and
  `untracked_buttons` reject `track_relay in ['0', '1']` and every LED write goes
  through those lists. Relay tracking and per-state colors are mutually exclusive
  on the big button; `track_relay0: '99'` switches it to blueprint control. Left
  as a documented decision for the user rather than silently resolved.
- `global_unavail_brightness: '10'` **is** written (it is a global, used by all
  tracked buttons), but has no visible effect while `global_unavail_state` is
  `'2'` (Always off) leaves the LED dark.

Rendered through the blueprint's own resolution templates: all 31 keys resolve to
declared inputs, every select value is a legal option, and the 4 tracked buttons x
3 states each resolve to their per-state brightness by day and to 2 after dark.

### Known limitations
- An override trigger whose `id` is literally `ButtonPress`, `StateTrigger` or
  `AutomationUpdate` will be ignored by the refresh branch.
- An empty Override Condition is vacuously true (Home Assistant's `and` checker
  returns `True` for an empty list). With all override values at their `No Change`
  default this is a no-op, so an unconfigured override cannot misbehave.
- Write amplification is unchanged: each re-evaluation writes 3 parameters x 5
  buttons with 100 ms delays, about 1.5 s of mesh traffic. Override triggers bound
  to a frequently-changing entity will repeat that. No debounce yet.

---

## 2026.10.02.2

UI usability changes only — no logic changes. Reorganized the settings layout
(switch/parameter inputs promoted to top level, panels start collapsed), updated
section icons and names, moved the 3-way switch panel under Night Time, and
reworded the Parameter 19/20 and Global Defaults labels.

---

## 2026.10.02.1

UI/UX cleanup and a new Global Defaults section. No change to the Z-Wave write
logic or triggers.

### Added
- **Global Defaults section.** New collapsed top-level section (placed under Switch
  params) with 9 dropdowns: color, behavior and brightness for the **on**, **off**
  and **unavailable** states. Defaults: on = Always on / Green / 6, off = On during
  daytime / White / 6, unavailable = Always on / Red / 6.
- **"Global Default" option on every per-button setting.** All 45 per-button
  state/color/brightness dropdowns (`off/on/unavail_*` for buttons 0–4) gained a
  `Global Default` option (value `global`), listed first, and it is now their
  default. A button falls back to the matching Global Default unless it sets its
  own value; explicitly chosen values override.

### Changed
- **Button panel titles** use a number-box icon in place of "Button N", keeping the
  title-cased position: Big Button (Relay), Top Left Button, Top Right Button,
  Bottom Left Button, Bottom Right Button.
- **Field names** stripped of the redundant "for button N" / position prefix, e.g.
  `Off behavior`, `Off color`, `Brightness when entity is off`, `Entity to Track`,
  `Pressed Once`.
- **"Multipress" renamed to "Scene Actions"** in all five sub-panel titles. Scene
  item names trimmed too (`Scene 3 - Held` → `Held`, `Scene 3 - Pressed 2x` →
  `Pressed 2x`). Internal section keys (`buttonN_multipress`) are unchanged.
- **Fallback logic.** Brightness maps (`map_b_off/on/unavail`) resolve `global` to
  the matching `g_*_brightness` before the name/number logic. The condition dict
  resolves `global` state/color to the matching `g_*` value. Added `g_*` variables
  bound to the new `global_*` inputs.
- Blueprint name is now `...(v2026.10.02.1)`.

### Compatibility
- Explicitly-set per-button values load unchanged. Any option previously left at its
  old hard-coded default will now follow Global Defaults after re-selecting this
  version; re-pick a specific value on a button to override.

---

## 2026.10.01.2

Per-state LED brightness now uses the firmware's own 1-10 numbers, and older
firmware is remapped in code.

### Changed
- **Brightness dropdowns (15 inputs).** `off_brightness0-4`, `on_brightness0-4` and
  `unavail_brightness0-4` now list the device scale, 1 to 10, instead of Low /
  Medium / Bright. The labels are `1 (dimmest)`, `3 (Low)`, `6 (Medium)`,
  `10 (Bright)` and plain numbers for the rest. The default is `6` (was `medium`).
  The selector is a fixed list (`custom_value: false`).
- **Firmware detection (`is_new_firmware`).** Auto-detect previously used the model
  name only (`ZEN32 800LR` or `ZEN35` = new scale). It now also reads the device's
  firmware version and treats a **ZEN35 as new-scale from 1.40** and a **ZEN32 800LR
  from 3.30**. A ZEN35 on 1.39 or earlier, or a ZEN32 800LR on 3.29 or earlier, is
  now treated as legacy. If the version can't be read, it falls back to the
  model-only check. The manual override (`new` / `legacy`) is unchanged. The version
  is parsed inside the template so `1.40` is not turned into `1.4`.
- **Brightness mapping (`map_b_off`, `map_b_on`, `map_b_unavail`).** The chosen level
  is written as-is on new-scale devices. On legacy devices it is remapped to the
  three hardware levels: **1-3 = Low (2), 4-7 = Medium (1), 8-10 = Bright (0)**.
- **Firmware input wording.** The auto-detect label and description now mention
  firmware version, not just model.

### Compatibility
- The old values `low`, `medium` and `bright` still work at runtime and map to 3, 6
  and 10. Existing automations load unchanged, but the UI dropdown won't show those
  values as selected, so re-pick them (3, 6, 10) when you next edit.
- A value that isn't a name or a number from 1 to 10 falls back to Medium (6). This
  includes the old raw legacy numbers: `0` becomes Medium, and `1` and `2` are now
  read as levels 1 and 2 of the 1-10 scale. On legacy firmware that makes a raw `1`
  Low (2), where before it was written as Medium (1).
- `99` ("no change") still passes through.

### Removed
- The 15 per-state brightness dropdowns previously allowed a typed exact value
  (`custom_value: true`) alongside Low / Medium / Bright. A numeric value was
  accepted if valid for the detected scale (1-10, or 0-2 on legacy); anything else
  fell back to Medium rather than being clamped. Helper variables `brightness_min`
  and `brightness_max` were added in an interim build and removed here.

---

## Changes from the original (v2026.9.5)

### Fixed
- **Unmapped brightness values were sent to the device unvalidated.** The original
  looked the value up in a Low / Medium / Bright map and, if it wasn't found, passed
  it through unchanged. A leftover legacy value such as `"0"` therefore reached a
  ZEN35 on the 1-10 scale, which has no level 0 and rejects it. This surfaced when a
  tracked entity (for example a shade group) changed to an "on" or unavailable state,
  and again after a restart. Values are now always mapped, validated or defaulted
  before being written.

### Metadata
- `source_url` removed so the Home Assistant re-import action does not overwrite this
  fork with the upstream file.
- Blueprint description gained a note describing the fork.

### Unchanged from the original
- All original inputs keep their keys. Night-time brightness (`night_led_brightness`)
  still uses Low / Medium / Bright and the original map. Triggers, mode, the Z-Wave
  write sequence (toggle, color, brightness with the 100 ms delay), the restart
  refresh and all other variables are unchanged.

> Historical: the night-time inputs described above were removed in
> **2026.10.03.1** and replaced by the Conditional Override.

---

## Testing

- Template logic rendered in Home Assistant's template engine: firmware boundaries
  (ZEN35 1.39 / 1.40, ZEN32 800LR 3.29 / 3.30), unknown version, manual overrides,
  and every brightness value and edge case on both scales.
- The exact template text from the blueprint was rendered against a real ZEN35 on
  firmware 1.40.0 (auto-detects as new scale).
- The blueprint YAML was parsed and structurally diffed against the original.
- v2026.10.02.1: YAML re-parsed after the Global Defaults / rename changes; verified
  all 45 per-button settings default to `global`, every `!input global_*` resolves to
  a defined input, and no "button N" / "Multipress" text remains in user-facing
  strings.
- v2026.10.03.1:
  - YAML re-parsed. All 127 declared inputs are referenced and all 127 `!input`
    references resolve — no orphans in either direction.
  - No references remain to any removed night variable or input.
  - `'99'` confirmed absent from all 18 behavior dropdowns; all 15 per-button
    override inputs present and defaulting to `global`; all 3 global override
    inputs defaulting to `'99'`.
  - The override resolution templates were extracted from the parsed action tree
    and rendered against 12 cases: global inheritance, per-button override of a
    single field, per-button value beating the global one, a button opting out,
    the legacy 1-3/4-7/8-10 remap at each boundary on legacy firmware, the old
    `low`/`medium`/`bright` names, and a garbage value. All passed.
  - Verified across every firmware/value combination that the `'99'` sentinel can
    never be written to the device as a brightness value.
  - Verified the override `choose` appears in all three write paths with the
    anchor reuse intact, and that each branch writes exactly 3 parameters.
  - Verified the trigger-merge entry is a single-key dict, which Home Assistant's
    `_base_trigger_list_flatten` requires in order to flatten it; an entry with an
    extra key such as `alias` would fail the base trigger schema. An empty
    `override_triggers` flattens to nothing and raises no error.
  - Confirmed against Home Assistant's `AndConditionChecker` that an empty
    condition list returns `True` rather than raising — the empty-Override-Condition
    default is therefore safe, and is a no-op because the override values all
    default to No Change.
  - Example config verified by rendering it through the blueprint's resolution
    templates: all 31 keys resolve to declared inputs, every value is a legal
    select option, the override trigger entries carry a `trigger:` key (not the
    UI editor's `options:` blob), the override condition is non-empty, and each
    tracked button/state pair dims to 2 only when the condition is true.
- Not yet verified end to end on the device. Check the Z-Wave JS log for rejected
  brightness writes after the first state change.
