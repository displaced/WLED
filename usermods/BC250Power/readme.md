# BC250Power (personal controller)

ESP32 usermod for Chris's BC-250 case. AI-assisted implementation based on this
project's [EXAMPLE usermod](../EXAMPLE/) and the supplied BC-250 handoff.
No BLE scanning, pairing, Home Assistant discovery, or suspend handling is included.

## Build

The local, gitignored `platformio_override.ini` selects `bc250_power`, extends
`esp32dev`, and loads only `BC250Power`. Build from the repository root:

```powershell
npm.cmd ci
npm.cmd run build
pio run -e bc250_power
```

If `pio` is not on PATH, use
`& "$env:USERPROFILE/.platformio/penv/Scripts/platformio.exe" run -e bc250_power`.
The firmware is `.pio/build/bc250_power/firmware.bin`. Build does not flash it.

To recreate the personal override:

```ini
[platformio]
default_envs = bc250_power

[env:bc250_power]
extends = env:esp32dev
custom_usermods = BC250Power
build_flags = ${env:esp32dev.build_flags}
  -D DATA_PINS=19
  -D LEDPIN=19
  -D BTNPIN=-1
  -D RLYPIN=-1
  -D IRPIN=-1
  -D AUDIOPIN=-1
  -D WLED_DISABLE_INFRARED
  -D WLED_DEBUG
```

The personal build enables WLED diagnostics (serial at **115200 baud** unless
the Wi-Fi logging host flag below is set). Messages
prefixed `BC250Power:` report initialization, state transitions, relay commands,
requested pulse durations, and debounced sense changes. Relay messages describe
the commanded output, not confirmation that the physical relay operated.
The normal WLED startup and diagnostic messages are also present.

After identifying the ESP's actual serial port, upload and then monitor it:

```powershell
$env:PYTHONUTF8 = '1'
$env:PYTHONIOENCODING = 'utf-8'
pio run -e bc250_power --target upload --upload-port COM_PORT
pio device monitor --port COM_PORT --baud 115200
```

Replace `COM_PORT` with the detected port, such as `COM5`. Close any existing
serial monitor before uploading so it releases the port. `Ctrl+C` exits the
monitor. With relay contacts disconnected, pressing the ESP's reset button after
opening the monitor lets you see the full startup log. Do not reset a controller
that is maintaining power to a running PC.

These are defaults, **not overrides of saved device settings**. Before connecting
relay contacts to the PSU or BC-250, check WLED LED, button, relay, IR, I2C, and SPI
settings. Use GPIO19 for LED data, no stock relay or button, and leave GPIO16,
GPIO17, GPIO22 and GPIO4 exclusively for this usermod. Do not load an audio usermod.
Core hardware is initialized before usermod setup (see `wled00/wled.cpp`), so a
conflict can already have driven a pin before this usermod detects it.

## Wi-Fi updates and logging

Browser OTA is already included. With the BC-250 shut down and the PSU relay off,
open `http://WLED-IP/update` (Config > Security & Updates > Manual OTA Update).
Unlock OTA and settings if required, then upload the custom build's
`.pio/build/bc250_power/firmware.bin`. Do not upload the bootloader, partition table,
or a stock WLED release: stock firmware omits this controller. Normal application
updates preserve saved settings. The ESP reboots after updating and initializes
both relays off. Keep its standby supply powered throughout the update.
See [WLED OTA documentation](https://kno.wled.ge/basics/getting-started/#software-update-procedure).

For Wi-Fi debug output, retain `-D WLED_DEBUG` and add this flag under the existing
`build_flags` in the local override, replacing the example with the receiving PC's
reserved LAN IPv4 address:

```ini
  -D WLED_DEBUG_HOST='"192.168.1.50"'
```

Rebuild and upload once to enable it. WLED sends debug messages to UDP port 7868
on that PC. The implementation is in `wled00/net_debug.cpp` and `wled00/wled.h`.
The current personal override targets Fenchurch at `192.168.50.146`; the ESP is
`loungepc-leds.home` (`192.168.50.216`). Reserve these addresses in DHCP.
Run the included receiver from the repository root:

```powershell
& "$env:USERPROFILE/.platformio/penv/Scripts/python.exe" -u usermods/BC250Power/monitor-wifi.py --esp WLED-IP
```

Replace `WLED-IP` with the ESP's actual IPv4 address. If Windows Firewall blocks
reception, allow inbound UDP 7868 from the ESP on your private LAN.
The host flag redirects WLED debug macros from serial to UDP; ROM boot messages,
early startup, and crashes before networking works are not available over Wi-Fi.
UDP is live best-effort output with no replay of missed logs. Serial remains useful
for recovery if firmware or networking fails. To return to serial debug output,
remove the host flag and rebuild. Runtime logging can be toggled with
`POST /json/state` containing `{"debug":false}` or `{"debug":true}`.

## Wiring and first use

| Function | Default GPIO | Behavior |
|---|---:|---|
| PSU relay | 16 | Active-high, maintained PS_ON# closure |
| BC-250 button relay | 17 | Active-high, timed button closure |
| Case button | 22 | Momentary switch to ground, internal pull-up |
| Power sense | 4 | Active-high, internal pull-down |
| ARGB data (WLED) | 19 | LEDs powered by switched PSU 5 V |

Both relays initialize inactive after successful pin allocation, even when the
controller is disabled. Pins remain reserved until reboot. Invalid, duplicate,
unassigned (`-1`), or occupied pins prevent **all** allocation and relay operations;
JSON reports `pinsReady: false` and `ERROR`. It does not take pins from other owners.

The software cannot hold relays off during ROM boot, reset, or loss of ESP power.
This board was verified active-high using COM/NO continuity; leave
`relay-active-low` unchecked. The original handoff's active-low assumption and
relay-input pull-up recommendation do not apply. Verify reset behavior with the
actual relay board's load contacts disconnected. Restarting/flashing the
ESP while the PC is running releases PS_ON# and may abruptly remove PC power.
Shut the PC down before restarting or updating the ESP.

The proposed TPMS1 pin 9 signal is **not verified by this implementation**. Verify
its voltage, reference ground, and behavior on the actual board. For automatic
cutoff it must become inactive after PC shutdown **while the PSU remains on**.
A signal that merely follows the PSU supply cannot detect that event. A signal
that drops during suspend would cause this implementation to cut power during
suspend as well; disable suspend while validating this first version.

Start with relay contacts disconnected:

1. Flash the personal build and check saved stock pin assignments.
2. In Config > Usermods > BC250Power, inspect the four pins in the table order.
   `enabled` and `automatic-cutoff` are both false initially.
3. Check `GET /json/state`: `pinsReady` should be true. Power sense updates even
   while disabled, allowing the input to be checked without energizing relays.
4. Set `enabled` true, save, and reboot with PC power off. Verify both relay
   outputs start off, the case button starts the PSU relay, and short/long presses
   operate the board-button relay as described below.
5. Verify sense on actual hardware. Only then set `automatic-cutoff` true and
   reboot with PC power off. With cutoff disabled, the PSU relay stays on even
   after sense disappears; this is intended for bench commissioning.

All usermod setting changes take effect **after reboot**. Saving settings does
not change pins, relay polarity, timings, or live PSU state. JSON flags
`rebootRequired` after a usermod settings update.
Malformed numeric/boolean values reject the whole update and set `configRejected`.
Malformed boot configuration prevents pin initialization. Missing fields retain
defaults (or the previously saved value), so future settings can be added safely.

## Behavior

- While enabled with valid pins, WLED master power follows the PSU relay,
  including when all preset IDs are zero. Power-on restores WLED's remembered
  brightness; power-off preserves it for the next start. Presets still select
  appearance and brightness, but conflicting preset/API/UI master power changes
  are corrected on the next controller loop. Segment settings remain independent.
  Disabled control or invalid pins do not enforce WLED master power.
- `OFF` + debounced case-button press asserts PS_ON# and enters `BOOTING`.
  The same held press is consumed until release; it cannot also force shutdown.
  A button held during ESP startup is ignored until released.
- This version assumes the BC-250 starts when 12 V is applied. It does not
  automatically pulse the board button at power-on. If needed, a subsequent
  short case-button press provides that pulse.
- Sense high confirms electrical power, not OS readiness. A `running` callback
  with stable sense high moves `BOOTING` (or timeout `ERROR`) to `RUNNING`.
- While the PSU is on, a short press pulses the board button on release. The OS
  decides the result; the pulse alone does not claim shutdown has started.
- Holding the case button for `long-press-ms` starts one `force-hold-ms` board
  button closure and enters `SHUTTING_DOWN`. Releasing the case button does not
  shorten that timed closure. Repeated requests do not extend an active pulse.
- Continuing that same press to **10 seconds total** releases both relays and
  enters `OFF`, regardless of power sense or `automatic-cutoff`. This is an
  emergency power cut and can interrupt disk writes. It cancels any remaining
  board-button pulse. Release is consumed without another pulse or power-on.
  The initial power-on press and a button held during ESP boot are exempt.
  After emergency cutoff, stuck-high sense does not change `OFF` to `ERROR`,
  but starting power again still requires sense to become inactive and a new press.
- A `shutting_down` callback enters `SHUTTING_DOWN`. Neither OS callback directly
  releases PS_ON#; late `running` callbacks cannot reverse `SHUTTING_DOWN`.
- Once sense has been observed high in the current power cycle, stable sense loss
  starts the cutoff grace period. If automatic cutoff is enabled and sense stays inactive
  through the grace period, both relays release and the state becomes `OFF`.
  Sense returning during the grace period prevents cutoff. Sense loss alone does
  not change the reported OS state during that period or apply a shutdown preset;
  the shutdown callback supplies that semantic event.
- A boot timeout enters `ERROR` and keeps PS_ON# asserted. If sense was never
  observed, the firmware never guesses that removing power is safe. Check wiring
  and the PC before using the emergency hold. There is no remote hard-cut API.
- A sense signal already high with PS_ON# off reports `ERROR`; it does not
  automatically energize the PSU relay. Stable sense low clears that condition.

Relay pulse timing uses `millis()` in the normal WLED loop. A busy/blocked loop can
lengthen pulses, and short button presses between loop samples can be missed.
This is not a real-time pulse generator: bench-check timing under your expected
LED/network load before attaching the PC button contacts.

## Settings

| Setting | Default | Accepted range / meaning |
|---|---:|---|
| `enabled` | false | Enable case-button control |
| `pin` | 16, 17, 22, 4 | PSU, board button, case button, sense |
| `relay-active-low` | false | Polarity of both relays; saved settings take precedence |
| `button-active-low` | true | Case button polarity; opposite internal pull |
| `sense-active-high` | true | Sense polarity; opposite internal pull |
| `automatic-cutoff` | false | Allow validated sense loss to release PS_ON# |
| `pulse-ms` | 250 | 50–1000 ms |
| `long-press-ms` | 2000 | 1000–10000 ms |
| `force-hold-ms` | 5000 | 3000–15000 ms |
| `sense-debounce-ms` | 100 | 20–2000 ms |
| `off-grace-ms` | 2000 | 100–60000 ms, after debounced sense loss |
| `boot-timeout-ms` | 120000 | 5000–600000 ms; error only, no forced cutoff |
| `booting-preset` | 0 | 0–250 |
| `running-preset` | 0 | 0–250 |
| `shutdown-preset` | 0 | 0–250 |
| `error-preset` | 0 | 0–250 |

Preset 0 leaves lighting appearance unchanged; master power still follows the PSU.
Other IDs use WLED's normal preset application and can refer to playlist presets.
Only transitions apply presets. No OFF preset is needed: the controller turns
WLED master power off when the PSU relay releases. Input GPIOs without
internal pull support require appropriate external resistors if reassigned.

## OS callbacks and status

Use the existing WLED JSON API on your trusted LAN. This usermod adds no separate
HTTP server or authentication mechanism; existing WLED endpoint policy applies.
POST the following to `http://WLED-IP/json/state` when the OS is ready:

```json
{"BC250Power":{"osEvent":"running"}}
```

At shutdown start, while networking is still available:

```json
{"BC250Power":{"osEvent":"shutting_down"}}
```

Example from the BC-250:

```sh
curl --fail --max-time 3 -H 'Content-Type: application/json' \
  -d '{"BC250Power":{"osEvent":"running"}}' http://WLED-IP/json/state
```

The callbacks are queued for the next loop iteration (the latest pending event
wins). An HTTP success is not confirmation of a state transition; check
`GET /json/state`. Events received while disabled, or without stable power sense
and an asserted PSU relay, do not change state. Retry `running` if it arrives
before the sense input stabilizes. Actual Bazzite/systemd integration is a later
hardware setup step; no services are installed by this repository change.

`GET /json/state` includes `BC250Power` with `state`, `psuOn`, `buttonRelayOn`,
`powerSense`, `pinsReady`, `enabled`, `rebootRequired`, and `configRejected`. `osEvent` is never
serialized, so saving a normal lighting preset does not record a command.
Do not deliberately put `osEvent` commands into lighting preset files.

## Bench acceptance checks

Before connecting relay contacts to the live PC/PSU, verify:

- Reset, cold boot, disabled control, and held-at-boot button leave relays off.
- Invalid, duplicate and occupied pins report `ERROR` without taking other pins.
- One start press asserts only the PSU relay; holding it does not force shutdown.
- With all preset IDs zero, WLED master power starts off, turns on with the PSU,
  and turns off on emergency or automatic cutoff. Conflicting master power from
  presets or the UI is corrected; colour/effect settings and brightness are retained.
- Sense high alone stays `BOOTING`; valid OS readiness changes to `RUNNING`.
- Short presses give one pulse; long presses give one hold, with no release pulse.
- A separate 10-second hold releases both relays with sense disconnected or held
  active, even with automatic cutoff disabled. Releasing before 10 seconds does
  not force PSU cutoff. Continuing to hold after cutoff does not restart power.
- Missing readiness gives `ERROR` without removing power; a later valid callback
  recovers to `RUNNING` if shutdown has not started.
- Shutdown callback alone keeps the PSU on. Sense loss before ever seeing power
  does not cut it. Brief sense glitches and returning sense cancel cutoff.
- Verified continuous sense loss cuts power only when automatic cutoff is enabled.
- Settings saves and lighting preset changes never operate relays by themselves.
- Pulse timing remains acceptable under LED updates, API requests and file writes.

Compilation and desktop tests cannot verify relay polarity, reset glitches,
TPMS1 behavior, or actual BC-250 power-button timing. Those require the bench checks.
