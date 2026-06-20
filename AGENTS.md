# AGENTS.md

Instructions for agents working in this Home Assistant configuration repo.

## Purpose

This repository is the YAML configuration for a Home Assistant install. The working copy is on a laptop. The live Home Assistant runtime is not here.

Treat this repo as configuration-as-code. Keep changes small, readable, and easy to reason about from the YAML alone.

## Operating Rules

- Follow the global agent instructions first.
- Do not invent abstractions for simple YAML changes.
- Prefer existing repo patterns over new layout or naming styles.
- Keep entity IDs stable unless the requested fix requires a rename.
- Do not edit `secrets.yaml` unless explicitly asked.
- Do not commit, push, or create branches unless explicitly asked.
- Do not run Home Assistant runtime checks on this laptop. This repo does not contain the HA runtime environment.

## Repository Shape

- `configuration.yaml` wires the repo together with Home Assistant includes.
- `automations.yaml` holds UI-style automations and most routine logic.
- `automation/` is included as merged manual automations.
- `script/` is included with `!include_dir_merge_named`.
- `scene/` is included with `!include_dir_merge_list`.
- `input_boolean/` is included with `!include_dir_merge_named`.
- `dashboards/` contains YAML Lovelace dashboards:
  - `home.yaml` for the main dashboard.
  - `mobile.yaml` for quick phone controls.
  - `ops.yaml` for admin and operational status.
- `data/entities.json` is an exported entity inventory. Use it as a reference, not as live state.

## Home Assistant Conventions

- Scenes must have explicit `id` fields when automations reference them by entity ID.
- Scene entity IDs are derived from scene `id`, for example `id: lr_good_night` becomes `scene.lr_good_night`.
- Prefer `scene.turn_on` for scene outcomes instead of direct `light.turn_on` or `light.turn_off` calls inside routine automations.
- Direct service calls are acceptable for non-scene side effects such as music, alarm state, input booleans, notifications, and fan boost.
- Keep conflicting routine helpers mutually exclusive. Turning on one routine should clear incompatible routine booleans.
- One-shot routine helpers such as `good_morning` and `good_night` should reset themselves after their work completes.
- Persistent routine helpers such as `wind_down` and `watching_tv` may remain on until manually cleared or reset.

## Adaptive Lighting

Adaptive Lighting can override scene brightness and colour if it is active while scenes are applied.

- Scene-based routines must call `script.adaptive_lighting_scene_mode` before `scene.turn_on`.
- Normal reset paths should call `script.adaptive_lighting_day_mode`.
- Bedtime-only behaviour should call `script.adaptive_lighting_sleep_mode` only when no scene routine is active.
- Do not add direct light service calls to fight Adaptive Lighting. Fix the adaptive mode sequencing instead.

## Dashboards

- Keep dashboard cards consistent with the existing Mushroom card style.
- When adding a normal light control, include it anywhere comparable room lights already appear.
- Main controls belong in `dashboards/home.yaml`.
- Phone-first quick actions belong in `dashboards/mobile.yaml`.
- Operational toggles, update status, and diagnostics belong in the Health view in `dashboards/home.yaml`.
- Do not turn dashboards into documentation pages. They should expose controls and state.

## Mobile And Phone Helpers

- iPhone helper booleans are triggers or mirrors for mobile-app state.
- Keep helper names and automation aliases clear enough to understand from Home Assistant UI.
- When a helper mirrors a sensor, keep the sync automation aligned with the source sensor states.
- Notification action IDs must match the handlers in `mobile_notification_actions`.

## Validation

Run local checks that work against this laptop checkout:

```sh
ruby -e 'require "yaml"; Dir.glob("**/*.yaml").sort.each { |path| YAML.load_file(path) }; puts "YAML parse passed"'
git diff --check
```

For scene references from `automations.yaml`:

```sh
ruby -e 'require "yaml"; automations = YAML.load_file("automations.yaml"); scene_ids = Dir.glob("scene/*.yaml").flat_map { |path| YAML.load_file(path).map { |s| "scene." + s.fetch("id") } }; refs = automations.flat_map { |a| a.to_s.scan(/scene\.[a-z0-9_]+/) }.uniq - ["scene.turn_on"]; missing = refs - scene_ids; raise "Missing scenes: #{missing.join(", ")}" unless missing.empty?; puts "Scene refs passed"'
```

For script references from `automations.yaml`:

```sh
ruby -e 'require "yaml"; scripts = Dir.glob("script/*.yaml").flat_map { |path| YAML.load_file(path).keys.map { |id| "script." + id } }; automations = YAML.load_file("automations.yaml"); refs = automations.flat_map { |a| a.to_s.scan(/script\.[a-z0-9_]+/) }.uniq; missing = refs - scripts; raise "Missing scripts: #{missing.join(", ")}" unless missing.empty?; puts "Script refs passed"'
```

Do not run `python3 -m homeassistant --script check_config -c .` here. It is the wrong environment. If full HA validation is needed, say it must be run on the Home Assistant host or through an explicitly provided remote command.

## Security

- Treat `secrets.yaml`, trusted networks, tokens, URLs, and notification endpoints as sensitive.
- Do not widen trusted networks, proxies, or auth providers without explicit instruction.
- Be conservative with Alexa exposure. `configuration.yaml` controls included domains.
- Avoid adding new externally reachable services or cameras to dashboards without checking whether that creates exposure.

## Done

A change is done when:

- The YAML is valid locally.
- Diff whitespace checks pass.
- Referenced scenes and scripts exist when touched.
- The behaviour matches the Home Assistant include structure.
- Any unavailable live validation is stated plainly.
