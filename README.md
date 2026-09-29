# Media Device Sync

**Power, input, and volume follow your media player.**

A Home Assistant blueprint that makes the gear downstream of a media player behave like one device. Works great with a **Sonos Port, Connect, or Zone Player** feeding a receiver or amplifier, and with smart plugs, switches, IR blasters and anything else you can control from Home Assistant.

Press play and everything turns on, switches to the right input and matches volume. Stop playing and it all shuts off after a delay. No remote, no input hunting, no amp left on all night.

[![Open your Home Assistant instance and show the blueprint import dialog.](https://my.home-assistant.io/badges/blueprint_import.svg)](https://my.home-assistant.io/redirect/blueprint_import/?blueprint_url=https%3A%2F%2Fraw.githubusercontent.com%2FYOUR_USER%2FYOUR_REPO%2Fmain%2Fmedia_device_sync.yaml)

## What it does

| When this happens | The blueprint does this |
|---|---|
| The source player starts playing | Turns on your switches/plugs, runs your custom start actions, turns on each receiver, waits for it to boot, selects the right input, and sets the volume |
| Volume changes on the source player (or on the receiver, in two-way mode) | Mirrors it to the other side, with a maximum-volume cap to protect your speakers |
| The source player has been stopped for the turn-off delay (default 5 minutes) | Turns off receivers still on your input, then switches/plugs, then runs your custom stop actions |
| You switch the receiver to a different input (TV, turntable, etc.) | Pauses the source player so it isn't playing to nobody |

## What can it control?

You can combine any of these:

- **Receivers and amplifiers exposed as media players.** Power, input selection and volume. Several receivers at once are supported.
- **Switches and smart plugs.** Any `switch`, `input_boolean`, `light`, `fan` or `remote` entity is turned on and off. Ideal for powered speakers or a vintage amp on a plug.
- **Anything else, via custom actions.** Run any Home Assistant actions on start and stop: IR blaster commands (Broadlink, Harmony), HDMI-CEC, scripts, scenes, notifications.

If your setup only has a smart plug and no receiver, leave the receiver empty and pick the plug. That works.

## Features

- **Reliable power-on.** Sends `turn_on`, waits for the receiver to report it's on, then waits a configurable settle time before selecting the input, so the input change isn't ignored while the receiver boots.
- **Startup volume and maximum volume.** Set a safe volume for when a receiver wakes, and a ceiling the blueprint will never exceed.
- **Won't hijack a receiver in use.** Optional: if the receiver is already on a different input (you're watching TV), leave it alone.
- **Safe volume sync.** Off, one-way, or two-way. Ignores tiny differences so two devices with different rounding don't bounce volume back and forth. Skips receivers that are off, unavailable, or on another input.
- **Enable switch.** Optionally only run while an `input_boolean` (or other helper) is on, for example a "Party mode" or "Guest mode" toggle.
- **Any source player.** Not limited to Sonos. Any `media_player` can be the source.
- **Grouped inputs.** The setup form is organized into collapsible sections, so basic setup stays simple.

## Requirements

- Home Assistant **2024.10.0** or newer
- A source `media_player` in Home Assistant (for Sonos, the [Sonos integration](https://www.home-assistant.io/integrations/sonos/))
- At least one thing to control: a receiver `media_player`, a switch/plug, or your own custom actions

## Installation

### One click

Click the **Import blueprint** badge at the top of this page and follow the prompts.

### Manual

1. Download `media_device_sync.yaml`.
2. Copy it to `config/blueprints/automation/<your_folder>/media_device_sync.yaml` in your Home Assistant configuration.
3. Open **Settings → Automations & Scenes → Blueprints** and confirm **Media Device Sync** is listed. If it isn't, check **Settings → System → Logs** for a blueprint error.

## Setup

1. Go to **Settings → Automations & Scenes → Create Automation → Use Blueprint**.
2. Choose **Media Device Sync**.
3. Fill in the inputs (below) and save.

**Important:** the receiver input name is **case-sensitive** and must exactly match an entry in the receiver's `source_list` attribute. Find it in **Developer Tools → States** by selecting your receiver. It might be `SONOS`, `Sonos`, or `AUDIO2` depending on the receiver. A mismatch is the most common reason nothing happens.

## Example setups

**Sonos Port into an AV receiver**
- Source player: your Sonos
- Receiver: your receiver's media player
- Receiver input name: the input the Port is plugged into (e.g. `SONOS`)

**Sonos Port into a powered amp on a smart plug**
- Source player: your Sonos
- Switches: the smart plug
- Leave receiver empty

**Amp controlled by an IR blaster**
- Source player: your Sonos
- Actions when playback starts: send the IR power-on command
- Actions when playback has stopped: send the IR power-off command

**Living room stack: receiver plus a subwoofer plug**
- Receiver: your receiver
- Switches: the subwoofer plug
- Optional: turn on *Do not take over a receiver in use* so it never grabs the receiver while you're watching TV

## Inputs

### Basic setup

| Input | Default | Description |
|---|---|---|
| **Source player** | none | The media player to follow. |
| **Receiver / media players to control** | empty | Receivers or amplifiers that appear as media players. Optional. |
| **Receiver input name** | blank | Exact, case-sensitive input name. Blank skips input selection. |
| **Switches to turn on and off** | empty | Smart plugs or other on/off entities. Optional. |

### Custom actions

| Input | Default | Description |
|---|---|---|
| **Actions when playback starts** | none | Any actions to run on start. |
| **Actions when playback has stopped** | none | Any actions to run on stop. |

### Volume

| Input | Default | Description |
|---|---|---|
| **Volume sync** | Two-way | Off, source player to devices, or two-way. |
| **Startup volume (%)** | 0 | Volume set when a receiver is turned on by this automation. 0 matches the source player. |
| **Maximum volume (%)** | 100 | Highest volume the blueprint will set on a receiver. Volume you set manually is not limited. |
| **Volume sync buffer time** | 1000 ms | How long a change must hold steady before it's copied. |
| **Volume tolerance** | 0.02 | Differences smaller than this are ignored (0.02 = 2%). |

### Behavior and timing

| Input | Default | Description |
|---|---|---|
| **Turn off delay** | 300 s | How long the source player must be stopped before everything turns off. |
| **Receiver power-on settle time** | 3 s | Extra wait after a receiver reports it's on, before selecting the input. |
| **Receiver power-on timeout** | 30 s | Maximum time to wait for a receiver to report it's on. |
| **Do not take over a receiver in use** | Off | Leave a receiver alone if it's on a different input. |
| **Pause source player when the receiver input changes** | On | Pause the source if you switch the receiver away from its input. |
| **Only run while these are on** | empty | Optional helpers that must all be on for playback start to run. |

## How it works

Five triggers, each handled by its own branch:

1. **Source player starts playing.** Runs the enable check, turns on switches, runs start actions, then for each receiver: turns it on if needed, selects the input, and sets the volume.
2. **Source player stays out of `playing` for the turn-off delay.** Turns off receivers that are still on your input, then switches, then stop actions. If a receiver is on a different input (you're watching TV), switches and stop actions are left alone.
3. **A receiver leaves your input.** Pauses the source player.
4. **Source player volume changes.** Copies it to each receiver, capped at the maximum volume.
5. **Receiver volume changes.** In two-way mode, copies it to the source player.

## Troubleshooting

**Nothing happens when the source plays.**
Check the receiver input name first (see the note under Setup). Then open the automation and check its **Traces** to see which step failed.

**The receiver turns on but stays on the wrong input.**
Increase **Receiver power-on settle time**. Some receivers ignore input changes for a few seconds after waking.

**Everything turns off while I'm still listening.**
Increase **Turn off delay**. Any pause longer than the delay counts as stopped.

**Volume keeps jumping around.**
Increase **Volume tolerance** or **Volume sync buffer time**, or set **Volume sync** to one-way.

**The blueprint doesn't show up after import.**
Home Assistant drops blueprints with schema errors. Look in **Settings → System → Logs** for `blueprint`. Make sure you're on 2024.10.0 or newer.

**I have other automations that control the same devices.**
Disable them. Two automations reacting to the same state change will conflict.

## Credits

Inspired by the original *Sonos Connect Sync* blueprint by [Qonstrukt](https://gist.github.com/Qonstrukt/ca1e761b2ec0a2d52fdb8c86490fbcbd). This version was rewritten for current Home Assistant and generalized to work with switches, custom actions, multiple receivers and any source player.

## License

Add a license of your choice here (MIT is a common default for blueprints).
