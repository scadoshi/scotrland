# Poly Voyager Focus 2 — No Audio / No Mic After Profile Switch

## What Happened

Switching the headset from `a2dp-sink` to `headset-head-unit` (required for mic use in Teams/browser) left audio silent and mic muted. WirePlumber had saved bad volume state and restores it every time the headset connects.

## Symptoms

- Headset connects and beeps fine
- No sound in browser (YouTube, Teams, etc.)
- Mic not picked up in calls
- `pactl list cards` shows correct profile (`headset-head-unit`)

## Fix

**1. Check the sink volume**
```bash
pactl list sinks | grep -A6 "bluez_output.38"
```
If `Volume` is low or `Mute: yes`, fix it:
```bash
pactl set-sink-volume bluez_output.38:5C:76:28:AD:77 65536 65536
pactl set-sink-mute bluez_output.38:5C:76:28:AD:77 0
```

**2. Check the mic**
```bash
pactl list sources | grep -A4 "bluez_input.38"
```
If `Mute: yes`, fix it:
```bash
pactl set-source-mute bluez_input.38:5C:76:28:AD:77 0
```

**3. Fix the saved state so it doesn't come back**
```bash
nano ~/.local/state/wireplumber/default-routes
```
Find these two lines and set `channelVolumes` to `1.000000`:
```
bluez_card.38_5C_76_28_AD_77:output:headset-hf-output={"channelVolumes":[1.000000], ...}
bluez_card.38_5C_76_28_AD_77:output:headset-output={"channelVolumes":[1.000000, 1.000000], ...}
```

## Why It Happens

WirePlumber saves hardware route volumes to `~/.local/state/wireplumber/default-routes` and restores them on every reconnect. If the headset volume was ever low (e.g. physical buttons), that low value gets saved and restored forever.

## Check If It's Back

```bash
grep "38_5C_76" ~/.local/state/wireplumber/default-routes
# channelVolumes should all be 1.000000
```
