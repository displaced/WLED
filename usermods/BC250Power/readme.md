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

- The main interface's **Info** panel has a **Power Button** control. It pulses
  relay 2 for `pulse-ms` (default 250 ms), independently of PSU and LED power.
  It works even when the PSU relay is off, provided the usermod is enabled and
  its pins are valid. It does not change the controller state or extend an
  active pulse. A running OS may interpret a short power-button press as sleep
  or shutdown; a sleeping machine may wake, according to its firmware/OS settings.
  The equivalent command is `{"BC250Power":{"pressButton":true}}` sent to
  `/json/state`. The command is never included in saved state or ordinary presets.
  Automatic sense cutoff remains unchanged: if sense falls during sleep, it can
  still remove PSU power. Verify the sleep sense behaviour before relying on wake.

- While enabled with valid pins, switching WLED master power from off to on
  requests PSU startup through the same checks as the case button. This includes
  normal UI/API commands and presets that switch master power on. Restoring the
  boot lighting state does not start the PSU.
- Switching WLED off only darkens the LEDs; it never requests PSU shutdown or
  pulses the BC250 button. LEDs can remain off while the computer runs.
- Starting the PSU with the case button restores remembered LED brightness.
  Emergency or automatic PSU cutoff switches the LEDs off. Presets may still
  select appearance, brightness and lighting power while the PSU is running.
  Disabled control or invalid pins do not enforce WLED master power.
- `OFF` + debounced case-button press asserts PS_ON# and enters `STARTING`.
  The same held press is consumed until release; it cannot also force shutdown.
  A button held during ESP startup is ignored until released.
- This version assumes the BC-250 starts when 12 V is applied. It does not
  automatically pulse the board button at power-on. If needed, a subsequent
  short case-button press provides that pulse.
- Stable sense high confirms electrical power and moves `STARTING` to `POWERED`.
  This reports only what the ESP can observe; it does not claim the OS has booted.
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
- Optional `running`, `sleeping`, and `shutting_down` callbacks enter `OS_RUNNING`,
  `SLEEPING`, and `SHUTTING_DOWN` respectively. The callbacks do not directly
  operate either relay, and late callbacks cannot reverse `SHUTTING_DOWN`.
- Once sense has been observed high in the current power cycle, stable sense loss
  starts the cutoff grace period. If automatic cutoff is enabled and sense stays inactive
  through the grace period, both relays release and the state becomes `OFF`.
  Sense returning during the grace period prevents cutoff. Sense loss alone does
  not change the reported OS state during that period or apply a shutdown preset;
  the shutdown callback supplies that semantic event.
- If sense does not become active before the boot timeout, the state enters
  `ERROR` and keeps PS_ON# asserted. The firmware never guesses that removing
  power is safe. Check wiring and the PC before using the emergency hold. A late
  valid sense signal recovers to `POWERED`. There is no remote hard-cut API.
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
| `boot-timeout-ms` | 120000 | 5000–600000 ms; sense timeout only, no forced cutoff |
| `powered-preset` | 0 | 0–250; applied on `POWERED` |
| `os-running-preset` | 0 | 0–250; applied on `OS_RUNNING` |
| `sleep-preset` | 0 | 0–250; applied on `SLEEPING` |
| `shutting-down-preset` | 0 | 0–250; applied on `SHUTTING_DOWN` |

Preset 0 leaves lighting appearance unchanged; master power still follows the PSU.
Other IDs use WLED's normal preset application and can refer to playlist presets.
Only transitions apply presets. `OFF`, `STARTING`, and `ERROR` do not apply a
preset. No OFF preset is needed: the controller turns WLED master power off when
the PSU relay releases. During migration, old `booting-preset`, `running-preset`,
and `shutdown-preset` values initialize `POWERED`, `OS_RUNNING`, and
`SHUTTING_DOWN` respectively; the next configuration write uses the new keys.
Input GPIOs without internal pull support require appropriate external resistors
if reassigned.

## OS callbacks and status

Use the existing WLED JSON API on your trusted LAN. This usermod adds no separate
HTTP server or authentication mechanism; existing WLED endpoint policy applies.
The electrical state does not require OS integration. Optionally POST the
following to `http://WLED-IP/json/state` when the OS is ready:

```json
{"BC250Power":{"osEvent":"running"}}
```

At shutdown start, while networking is still available:

```json
{"BC250Power":{"osEvent":"shutting_down"}}
```

Immediately before entering sleep:

```json
{"BC250Power":{"osEvent":"sleeping"}}
```

Example from the BC-250:

```sh
curl --fail --max-time 3 -H 'Content-Type: application/json' \
  -d '{"BC250Power":{"osEvent":"running"}}' http://WLED-IP/json/state
```

The callbacks are queued for the next loop iteration (the latest pending event
wins). An HTTP success is not confirmation of a state transition; check
`GET /json/state`. Events received while disabled, or without stable power sense
and an asserted PSU relay, do not change state. Stable sense enters `POWERED`;
`running` then refines that state to `OS_RUNNING`, and a later `running` event
returns `SLEEPING` to `OS_RUNNING`. A stable-sense `running` event also returns
`SHUTTING_DOWN` to `OS_RUNNING`, which handles an OS reboot where power never
drops. A `sleeping` event cannot reverse shutdown. A sleep event does not disable
automatic cutoff, so the sense input must remain active during sleep if PSU power
is to be retained. OS integration is optional. On a Linux system using systemd,
install the supplied hooks with:

```sh
sudo bash ./usermods/BC250Power/install-os-integration.sh
```

The default WLED address is `http://loungepc-leds.home`. Pass another HTTP origin
as the first argument when required:

```sh
sudo bash ./usermods/BC250Power/install-os-integration.sh http://192.168.50.237
```

The installer creates a notifier in `/usr/local/sbin`, its configuration in
`/etc/bc250-wled.conf`, and two systemd services. One reports `running` after the
network comes online and `shutting_down` before it goes offline. The other reports
`sleeping` before suspend and `running` after resume. Startup notification retries
to allow the network and WLED time to become reachable. Sleep and shutdown use
only two short attempts, and notification failure always returns success so an
unavailable WLED device cannot prevent an OS transition.

Re-run the installer to update the destination or installed files. The services
can be inspected with:

```sh
systemctl status bc250-wled-online.service bc250-wled-sleep.service
journalctl -u bc250-wled-online.service -u bc250-wled-sleep.service
```

`GET /json/state` includes `BC250Power` with `state`, `psuOn`, `buttonRelayOn`,
`powerSense`, `pinsReady`, `enabled`, `rebootRequired`, and `configRejected`. `osEvent` is never
serialized, so saving a normal lighting preset does not record a command.
Do not deliberately put `osEvent` commands into lighting preset files.

## Xbox controller learning

Passive scanning requests a 10 ms window every 200 ms (5% nominal duty cycle),
reduced from 40 ms every 80 ms to give Wi-Fi more opportunity to use the shared
radio. The scanner also requests the SDK's Wi-Fi-preferred coexistence policy;
controller detection can tolerate lower Bluetooth priority. Actual airtime is
controlled by the SDK coexistence scheduler. Detection can take longer or miss a
brief advertisement burst; close-range controller detection latency and Wi-Fi
responsiveness must both be checked on the device.

Scanning runs while the PSU relay is off and while the OS reports `SLEEPING`.
PSU startup and the `running` callback pause reception and cancel any learning
session. PSU shutdown and the `sleeping` callback resume scanning automatically
with the existing fifteen-second startup guard. Turning only the LEDs off does
not resume scanning. The Bluetooth stack stays initialized, but discovery is
stopped while the BC250 is awake.

Before starting Bluetooth scanning, open **Config > WiFi Setup**, uncheck
**Disable WiFi sleep**, save, and reboot with the PSU off. The stable Arduino 2
SDK requires Wi-Fi modem sleep for Bluetooth coexistence. With sleep disabled,
starting Bluetooth after joining Wi-Fi reproduced an abort in
`coex_core_enable`; enabling sleep eliminated that failure on the spare ESP32.
The usermod does not currently enforce this prerequisite automatically. Keep
Wi-Fi sleep enabled even after stopping scanning, since the Bluetooth stack
remains initialized until reboot.

The ESP32 build includes a passive NimBLE scanner, disabled by default. Open
WLED's **Info** panel and use **Start scanning**, then **Learn**. Keep the target
controller off for the five-second baseline, then power it on normally when the
instruction changes. After the thirty-second search, select **Save** beside its
address and enter a name. The panel refreshes at the phase boundaries; **Refresh**
also retrieves current results. Confirmation expires after sixty seconds.

Up to eight controllers can be saved. Each has **Rename**, **Enable/Disable**, and
**Delete** controls. Repeat Learn for additional controllers. Candidates must
advertise gamepad appearance, Microsoft's company ID and the HID service together,
with at least three observations spanning 200 ms. Baseline devices and saved
controllers are excluded. There is no automatic selection when several appear.

Saved enabled controllers request the normal PSU startup sequence when the PSU is
off. While the state is `SLEEPING`, detection instead pulses relay 2 once for the
configured short-press duration, leaving the PSU relay and LEDs unchanged. Other
powered states do not scan. The power controller must be enabled with valid pins.
The separate **Power Button** remains available for manual wakeup. Learning and
configuration changes do not request startup or wakeup. Detection is inhibited
for fifteen seconds after scanning starts, list edits, PSU shutdown, and entry
into sleep.

A fifteen-second gap in advertisements rearms detection. This observes an
advertising appearance, not a guaranteed physical power-button press: reconnection
can also advertise, and a connected controller can stop advertising. BLE identity
and classification are not authentication. Controllers using rotating private
addresses may need relearning; this does not pair with or take over the controller.

The list lives in WLED's normal configuration, so configuration backups include
it. `controller-scan` and the eight `controller1` ... `controller8` settings groups
apply live; an empty address represents an unused slot. Hardware settings still
require reboot. The prototype's NVS list is not automatically imported into WLED.
Saving from the Info panel queues WLED's usual configuration write; wait for it
to complete before disconnecting power, and verify the list after reboot.

The sleeping LED preset is stored as `sleep-preset`. Older configurations using
`sleeping-preset` migrate automatically; the shorter name avoids WLED's generic
settings-page heuristic that mistakes any numeric field containing `pin` for a
GPIO selector. The settings page fetches WLED's preset list and displays all four
preset fields as dropdowns containing the preset ID, name, and optional quick-load
label. The numeric preset ID remains the persisted value.

The JSON API reports `BC250Power.controllers` in `/json/state`, including the
current `revision`, phase (0 idle, 1 baseline, 2 search, 3 confirmation), entries
and candidates. POST commands use the separate singular `controller` object:

```json
{"BC250Power":{"controller":{"action":"learn","revision":3}}}
{"BC250Power":{"controller":{"action":"save","revision":4,"slot":1,"name":"Living room"}}}
```

Always use the latest reported revision, not the example numbers. Other actions
are `scanOn`, `scanOff`, `cancel`, `rename`, `enable`, `disable`, and `delete`.
Slots are one-based; Save refers to a candidate slot, other slot operations to a
saved controller. Names accept 1–24 printable ASCII characters excluding quotes
and angle brackets. Invalid/stale commands leave the list unchanged and report a
message. These commands use the existing WLED JSON endpoint and its access policy.

The parser and learning state machine are shared with
`experiments/xbox-detector/src/LearningTests.h` (26 prototype self-tests). The
scanner uses the stable toolchain's [bundled NimBLE stack](https://docs.espressif.com/projects/esp-idf/en/v4.4.8/esp32/api-reference/bluetooth/nimble/index.html)
in passive observer mode, without an additional Bluetooth library. No WLED
web-page or core code changes are required for these controls.

The stable Arduino 2 build must use `WLED_USE_SHARED_RMT`, already included in
`tools/bc250-platformio.ini`. The alternative high-priority LED RMT driver caused
a Bluetooth interrupt crash during bench testing; the shared RMT build scanned
successfully. The firmware still fits the existing 1.5 MB OTA application slot.

Bench validation on a spare ESP32 with one Xbox model 1914 verified learning,
explicit saving, rename, enable/disable, invalid and stale command rejection,
HTML escaping, and persistence across reboot. A learned controller started the
simulated PSU without pulsing relay 2; switching LEDs off kept the PSU on.
These checks used serial JSON with the Wi-Fi access point active. Operation with
the actual relay hardware, LED load and station Wi-Fi still needs deployment
testing; only one physical controller was available.

## Bench acceptance checks

Before connecting relay contacts to the live PC/PSU, verify:

- Reset, cold boot, disabled control, and held-at-boot button leave relays off.
- Invalid, duplicate and occupied pins report `ERROR` without taking other pins.
- One start press asserts only the PSU relay; holding it does not force shutdown.
- With all preset IDs zero, WLED master power starts off; switching it on starts
  the PSU. Switching it off while running leaves both relay commands unchanged
  and stays dark across subsequent controller loops. Switching it back on restores
  lighting without pulsing the board button. A case-button start lights the LEDs.
- Emergency or automatic cutoff darkens the LEDs without restarting the PSU.
  Boot brightness and boot presets must not start the PSU after ESP reset.
- Sense high changes `STARTING` to `POWERED` without an OS callback.
- With stable sense, `running` changes `POWERED` or `SLEEPING` to `OS_RUNNING`,
  and `sleeping` changes `POWERED` or `OS_RUNNING` to `SLEEPING`.
- With stable sense throughout an OS reboot, `shutting_down` followed by `running`
  changes `SHUTTING_DOWN` back to `OS_RUNNING`.
- In `SLEEPING`, scanning resumes after its fifteen-second guard. A newly seen
  learned controller pulses only relay 2 once; the PSU relay stays asserted.
  The subsequent `running` callback stops scanning without another pulse.
- Short presses give one pulse; long presses give one hold, with no release pulse.
- A separate 10-second hold releases both relays with sense disconnected or held
  active, even with automatic cutoff disabled. Releasing before 10 seconds does
  not force PSU cutoff. Continuing to hold after cutoff does not restart power.
- Missing sense gives `ERROR` without removing power; a later valid sense signal
  recovers to `POWERED` if shutdown has not started.
- OS callbacks alone keep the PSU on. Sense loss before ever seeing power
  does not cut it. Brief sense glitches and returning sense cancel cutoff.
- Verified continuous sense loss cuts power only when automatic cutoff is enabled.
- Settings saves and colour/effect changes do not start power. A non-boot preset
  that switches WLED master power from off to on can start the PSU.
- Pulse timing remains acceptable under LED updates, API requests and file writes.

Compilation and desktop tests cannot verify relay polarity, reset glitches,
TPMS1 behavior, or actual BC-250 power-button timing. Those require the bench checks.
