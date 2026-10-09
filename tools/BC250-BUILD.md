# Building BC250 firmware against WLED releases

Run from the repository root using Windows PowerShell, Git, Node/npm and
PlatformIO (the VS Code installation under your user profile is detected).

```powershell
# Build the pinned upstream release with the current local usermod:
powershell -NoProfile -ExecutionPolicy Bypass -File tools/build-bc250.ps1

# Query GitHub for its latest non-prerelease and build it:
powershell -NoProfile -ExecutionPolicy Bypass -File tools/build-bc250.ps1 -Latest

# Select a particular release:
powershell -NoProfile -ExecutionPolicy Bypass -File tools/build-bc250.ps1 -Tag v16.0.1

# Prepare sources and check integration without downloading build dependencies:
powershell -NoProfile -ExecutionPolicy Bypass -File tools/build-bc250.ps1 -PrepareOnly
```

VS Code's **Terminal > Run Task** also offers **BC250: Build pinned release**
and **BC250: Build latest stable**.

## What the script does

1. Reads the tag and expected upstream commit from `bc250-release.json`, unless
   an explicit tag or latest release was selected. Latest always fetches from
   the official WLED repository. Explicit tags use an existing local tag if
   available; otherwise they are fetched from that repository.
2. Exports upstream sources into a fresh directory under
   `build_output/bc250/<tag>-<timestamp>-<unique suffix>/source`. It does not
   switch branches, reset files, or require committing local usermod edits.
3. Copies `BC250Power.cpp`, `ControllerLearning.h`, `ControllerScanner.h`,
   `library.json`, and the tracked
   `bc250-platformio.ini` configuration. The gitignored root override is not
   used. Edit the tracked configuration to change future builds, including
   the UDP log destination (currently `192.168.50.146`). Device Wi-Fi credentials
   and runtime usermod settings are not embedded in this file.
4. Adds the usermod ID and pin-owner entry only to the exported sources.
   Occupied ID 59 or changed integration anchors stop the build for review.
5. Runs `npm ci`, `npm test`, `npm run build`, and PlatformIO's `bc250_power`
   build sequentially. A failure stops the script. No successful artifact is
   published until all these steps pass.
6. Places `BC250-<tag>.bin` and `manifest.json` beside the source directory.
   The manifest records the upstream commit, hashes of custom inputs, and
   the successful firmware's SHA-256. Failed builds retain their source tree
   for diagnosis; a prepared manifest is not evidence of a successful build.

WLED 16.0.1's web tests require a generated file to be less than 500 ms old
when the build finishes. This fails on slower Windows builds. Where that exact
assertion exists, the script adjusts the exported test to require the file's
modification time to be at or after the generating command's start instead.
Tests still verify actual regeneration. The manifest records this test-only
correction; the checkout's tests and firmware logic are not modified.

The script never flashes, changes partitions on a device, or modifies device
settings. Use the resulting application `.bin` for a separately controlled OTA
update. Shut down the BC250 before updating its power controller. Keep backups
and the previously working firmware, and verify pins, polarity, startup relay
states, force-off, sense cutoff, LED power and logging after migration.

## Updating the pinned release

Build with `-Latest`, test the firmware on the hardware, then copy its `tag` and
`upstreamCommit` from the manifest into `bc250-release.json` (`commit` field).
Latest builds do not silently advance this pin. The initial pin is v16.0.1;
compilation is not a substitute for hardware validation.

Every run creates a fresh source directory, so first builds can be slow and
use substantial disk space. PlatformIO's shared downloaded toolchains remain
cached. Old run directories may be removed manually after retaining needed
firmware and manifests. Upstream dependency declarations may include mutable
references: the manifest identifies source inputs, not a guarantee of
bit-for-bit reproducibility of all transitive dependencies.

AI assistance was used to implement this workflow.

## Experimental Wi-Fi task on core 1

Run `powershell -NoProfile -ExecutionPolicy Bypass -File tools/build-bc250.ps1 -WifiCore1`
to test Wi-Fi/Bluetooth task separation on the classic dual-core ESP32.
This requires the installed Tasmota framework version `2.0.18+sha.8ba0dc0`
or the verified experimental package from a previous run.
The script makes a private framework copy in the build directory and verifies
the original `WiFiGeneric.cpp` hash before setting `cfg.wifi_task_core_id = 1`
at Wi-Fi initialization. Bluetooth controller and NimBLE remain on core 0.
The precompiled ESP-IDF libraries are unchanged; this uses the SDK's documented
[Wi-Fi initialization field](https://docs.espressif.com/projects/esp-idf/en/v4.4.8/esp32/api-reference/network/esp_wifi.html).

The experimental JSON state adds `BC250Power.controllers.taskCores`, reporting
the affinity of the `wifi`, `btController`, and `ble` (NimBLE host) tasks (`-1` when a
task is absent). These diagnostics are excluded from normal builds. The scan
timing remains 10 ms every 200 ms. Wi-Fi and the WLED Arduino loop now share core
1, so LED timing and web responsiveness both need validation. The radio is still
shared regardless of CPU affinity. This option does not flash either device.
Omit `-WifiCore1` to return to the standard framework build.
PlatformIO selects packages in its shared cache, so switching build types may
reinstall the framework. The preparation script never patches that shared copy
in place; subsequent experimental builds validate and copy the cached experiment.

Bench results on the spare ESP32 (2026-10-09), with the same 5% scan timing:

| Firmware | Successful HTTP requests | Median successful response | Maximum successful response |
| --- | --- | --- | --- |
| Standard Wi-Fi core 0 | 28/30 | 1.672 s | 5.125 s |
| Experimental Wi-Fi core 1, final diagnostics | 30/30 | 0.188 s | 0.328 s |

Runtime task queries confirmed Wi-Fi on core 1 and both Bluetooth controller and
NimBLE host on core 0. Power-state tests also verified that PSU startup cancels
learning and stops scanning, LED-off does not stop the PSU, and scanning resumes
after reboot with the PSU off. These are short sequential bench measurements,
not proof of long-term reliability or performance on the main installation.
Turning the learned Xbox controller on normally also started the spare's
simulated PSU, stopped scanning, and left the power-button relay inactive.
OTA remained unreliable during the experiment; the final spare upload used
serial and passed flash hash verification. Main-board deployment is not yet tested.
