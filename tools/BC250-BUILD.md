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
