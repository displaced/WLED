# Xbox controller detection experiment

Standalone PlatformIO project for a spare classic ESP32 (`esp32dev`). It does
not contain WLED or drive any GPIOs. Output is serial at 115200 baud; this avoids
assuming that the spare board has an LED on a particular pin.

Initial test controller: Xbox Wireless Controller model 1914 (FCC ID C3K1914),
currently paired to the BC250 over Bluetooth. Normal power-on advertisements were observed twice at `a8:8c:3e:66:b5:a4`,
public address type 0. Firmware version, privacy behaviour over longer periods,
and connection/sleep behaviour still need validation.

This first experiment listens passively for BLE advertisements. It never pairs,
connects, or changes the controller's existing pairing. Older Xbox controller
firmware can use Bluetooth Classic; Xbox Wireless adapter traffic is a different
protocol. BLE visibility during ordinary power-on/reconnection is something to
measure, not a guarantee. See [Bluepad32 controller notes](https://bluepad32.readthedocs.io/en/latest/supported_gamepads/).

Open this folder as a separate PlatformIO project, or run from the WLED root:

```powershell
& "$env:USERPROFILE/.platformio/penv/Scripts/platformio.exe" run -d experiments/xbox-detector
# Substitute the spare board's port, not the relay controller's port:
& "$env:USERPROFILE/.platformio/penv/Scripts/platformio.exe" run -d experiments/xbox-detector -t upload --upload-port COMxx
& "$env:USERPROFILE/.platformio/penv/Scripts/platformio.exe" device monitor -p COMxx -b 115200
```

## Multi-controller learning

Up to **eight** named controllers are saved in NVS as a versioned list. Each
entry has an address, address type and enabled flag. Slots are numbered 1-8;
deleting a controller leaves the other slot numbers unchanged.

Learning requires all three fields in the same well-formed advertisement:
Gamepad appearance `0x03C4`, Microsoft company ID `0x0006`, and HID service
`0x1812`. This is an advertisement classification, not authenticated identity.
It deliberately excludes devices that omit those fields; it does not fall back
to choosing arbitrary new devices. Names and RSSI are not used as identity.

Send newline-terminated commands at 115200 baud:

| Command | Effect |
| --- | --- |
| `learn` | Start a five-second baseline with the target controller OFF |
| `candidates` | Show current candidates, packet counts and signal strength |
| `save 1 Living room` | Confirm candidate 1 using the name `Living room` |
| `cancel` | End learning without changing the saved list |
| `list` / `status` | Show saved controllers / learning state and dropped packets |
| `rename 1 Chris` | Rename saved slot 1 |
| `enable 1` / `disable 1` | Set whether slot 1 emits detection events |
| `delete 1` | Delete saved slot 1 immediately |
| `selftest` | Run isolated logic tests; does not alter stored settings |

After `learn`, wait for **POWER ON** before switching the controller on normally.
The search lasts 30 seconds, then the list freezes for confirmation. Only devices
absent from the baseline and not already saved qualify. A candidate needs at
least three classified observations spanning 200 ms. Multiple candidates are
listed individually; the strongest is never silently selected. Use `save` during
the following 60-second confirmation window. Nothing is saved automatically.
Cancel, timeout, insufficient packets, overflow, duplicate identity and a full
list cannot replace existing controllers. Each learn/save session adds one entry;
repeat to add more controllers. Printable names have a 24-character limit (no
quotes or angle brackets). Overlong serial commands are rejected entirely.

Saved enabled controllers emit `DETECTED` on observed appearance. Learning
suppresses these events; saving a candidate or renaming a controller does not
fabricate a power-on event. After 15 seconds without classified observations,
the controller rearms. Silence does **not** prove power-off: a connected
controller may stop advertising. This remains a serial-only experiment and is
not yet an approved PC power-on policy.

The old single-target setting migrates only after its address is observed with
the full controller classification, supplying its address type. It is named
`Original controller`. The old key is removed after the new list saves. Invalid
stored data disables writes rather than silently overwriting it.

### Test coverage

`selftest` exercises the actual parser and learner against the captured Xbox
packet, malformed/truncated packets, non-gamepads, missing classification fields,
baseline exclusion, observation threshold, multiple candidates, explicit save,
duplicate rejection, address types, timeout/cancel, timer rollover and capacity
limits. Test objects are isolated from the live learner and NVS.

Verified on the spare ESP32: firmware compilation/upload, all 26 self-tests,
migration of the original controller, rename/enable/disable commands, persistence
after reboot, real learning timers, rejected invalid saves, and cancellation
preserving the list. Multiple candidates are covered by synthetic observations;
learning a second physical controller still requires a live test.

### WLED adapter

The canonical dependency-free parser and learner now live in
`usermods/BC250Power/ControllerLearning.h`. This prototype includes that same
header, so its 26 self-tests exercise the production learning state machine.
The WLED adapter, UI controls, JSON commands and persistence format are described
in `usermods/BC250Power/readme.md`. Prototype NVS and WLED configuration remain
separate; flashing WLED does not import the prototype's saved list.

Test repeated normal power-ons with the PC off, sleeping, and running. Record
whether the same address appears and whether detection happens before connection
to the PC. A later WLED integration needs evidence of stable identity and normal
power-on visibility, false-trigger handling, and Wi-Fi/BLE coexistence testing.
There is deliberately no connection from this prototype to the BC250 power API.

API reference: [Espressif BLE GAP](https://docs.espressif.com/projects/esp-idf/en/v4.4.8/esp32/api-reference/bluetooth/esp_gap_ble.html).
