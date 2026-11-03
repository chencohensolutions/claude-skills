---
name: android-emulator
description: Run an Android app on the owner's phone or the project's emulator — start and stop the emulator, pick the device, build, install, open the app and take screenshots. Use whenever a task needs the Android app running, on a device or emulator, or a screenshot of it, or when the owner asks to start, stop or check the emulator.
---

# Android: phone or emulator

## The project's notes

Each project names its emulator and app in **`.claude/android-emulator.md`**: the AVD name, the app id (and the UI
lab's, if it has one), and the project's own run scripts. Read it first. Without it, run
`$ANDROID_HOME/emulator/emulator -list-avds` and pick the AVD named after the project; if none is, ask the owner,
then write the notes.

Tools: `sdk="${ANDROID_HOME:-$HOME/Library/Android/sdk}"`, `adb="$sdk/platform-tools/adb"`. Gradle needs a JDK:
`JAVA_HOME`, else Homebrew's `openjdk@21` (`/opt/homebrew/opt/openjdk@21/libexec/openjdk.jdk/Contents/Home`), else
the one bundled with Android Studio (`/Applications/Android Studio.app/Contents/jbr/Contents/Home`).

## Which device

The owner's phone comes first: `adb devices` lists it as a serial that doesn't start with `emulator-` (USB or
wireless debugging). With no phone, use the project's emulator. The owner can ask for either; `DEVICE=phone`,
`DEVICE=emulator` or `DEVICE=<serial>` is how the project scripts take that choice.

Always pass `-s <serial>` to adb once a device is picked, since another may connect meanwhile.

## The emulator

Start it only when it's needed and stop it when the task is done, since it holds several GB of memory. Use the
project's `scripts/emulator.sh` when it has one; otherwise this skill's `scripts/emulator.sh` (in the skill's base
directory, shown when the skill loads), with `AVD=<name>` from the notes:

    AVD=<name> <skill dir>/scripts/emulator.sh up       # headless; waits until Android has booted, prints the serial
    AVD=<name> <skill dir>/scripts/emulator.sh status
    AVD=<name> <skill dir>/scripts/emulator.sh down     # saves the quick-boot snapshot, so the next up takes seconds

`up` shuts down other running emulators first, because scripts and adb pick the first one (`KEEP_OTHERS=1` keeps
them); `WINDOW=1` shows its window, which the owner may want to watch. Run `up` in the background when the build
can go on meanwhile. If it fails, its log is in `${TMPDIR:-/tmp}/emulator-<avd>.log`.

## Build, install, open

Prefer the project's scripts (the notes name them: a run script, the UI lab script). Done by hand:

1. Build only the device's ABI when the project supports it (`-Pabi=$(adb -s <serial> shell getprop
   ro.product.cpu.abi)`, or `ABI=` for the project's script): native libraries make a full APK slow to send over
   wireless debugging.
2. `adb -s <serial> install -r <apk>`. `-r` keeps the app's data; never uninstall or `pm clear` the owner's app on
   their phone without asking, since it holds their real data.
3. Open it with `adb -s <serial> shell am start --user current -S -f 0x10008000 -n <app id>/<activity>`:
   `-S` stops the old process so the new build runs, `--user current` is needed on a phone with a Secure Folder,
   and the flags (new task, clearing the old one) stop Android resuming an earlier screen.

## Look at it

- Screenshot: `adb -s <serial> exec-out screencap -p > <file>.png`, then look at the file. All black from the phone
  means it is locked: ask the owner to unlock it.
- Tap and type for a flow: `adb -s <serial> shell input tap <x> <y>`, `input text <text>`, `input keyevent
  KEYCODE_BACK`; find coordinates with `adb -s <serial> exec-out uiautomator dump /dev/tty`.
- Logs: `adb -s <serial> logcat -d --pid=$(adb -s <serial> shell pidof <app id>)`, or `-s <tag>` for a tag. Crash:
  `adb -s <serial> logcat -d -b crash`.
- Dark mode and locale for edge cases: `adb -s <serial> shell cmd uimode night yes|no`; change the locale on the
  emulator only, not on the owner's phone.

Tell the owner which device each result came from. Put back anything you changed on their phone (dark mode, for
instance) when done, and stop the emulator if you started it.
