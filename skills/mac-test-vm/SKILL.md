---
name: mac-test-vm
description: Launches, screenshots and clicks through Mac app builds inside a headless macOS test VM with the testvm command, so testing never takes over the user's screen. Use it every time you're about to open, relaunch, screenshot, click or UI-test a Mac app you built (Xcode, SwiftPM or Electron), instead of running `open` on this Mac. Also use it when the user mentions the test VM or testvm, or says agents keep taking over their screen.
---

# Testing Mac apps in the test VM

The user keeps working on this Mac while you build their apps. When you open an app here, it steals the focus and interrupts them. So every launch, screenshot and click happens in a macOS virtual machine, through the `testvm` command.

The VM is called `mac-test` and runs under [Tart](https://tart.run), with no window and its own screen. Nothing shows up on the user's screen.

## Rules

- Never run `open` on an app you built, and never quit or relaunch the user's copy. Use `testvm open`.
- Build on this Mac as usual. Only the finished `.app` goes to the VM.
- XCUITest UI tests take over the screen too. Drive the app in the VM instead.
- When you're done, tell the user where the build is, so they can open it themselves.

If `testvm` isn't installed, point the user to https://github.com/flaviocopes/testvm.

## Workflow

Start with a note that says what you're about to test. It goes in the log, and VM Monitor shows it next to your chat:

```bash
testvm note 'Checking the new Sort by usage menu in Skillscout'
```

Then build, and open the build in the VM. `testvm` starts the VM when it's not running, which takes about 15 seconds.

```bash
xcodebuild -project Skillscout.xcodeproj -scheme Skillscout -configuration Debug -derivedDataPath build build
testvm open build/Build/Products/Debug/Skillscout.app
```

`testvm open` copies the app to `~/Apps` in the VM, quits the old copy, launches the new one and waits for a window. Launch arguments go after the path. For a SwiftPM app, open the `.app` its bundle script produces, not the bare executable.

Take a screenshot of the app's window:

```bash
testvm shot Skillscout
```

It prints the path of a PNG in `/tmp/testvm`, which you can look at. Leave out the name to capture the whole VM screen, for menu bar apps, menus and alerts outside the window.

To see what's in the window without guessing from pixels, list its controls. Each one comes with its text and the point to click:

```bash
testvm ui Skillscout
```

```text
WINDOW "Skillscout"
AXStaticText "Missing somewhere"  center 255,200
AXPopUpButton [pop up button] = Newest first  center 855,113
AXButton "Find repeated tasks" (disabled)  center 976,113
AXTextField [search text field] = release  center 1171,114
```

A name in quotes is the control's label. Square brackets mean it has no label, so you get its type instead. Coordinates are screen points, the same ones `click` takes. Then interact with it:

```bash
testvm click 255 200
testvm click 1171 114
testvm type 'release'
testvm key cmd+a
testvm key return
```

`key` takes `cmd`, `shift`, `option` and `ctrl` with one character, or a named key: `return`, `esc`, `tab`, `space`, `delete`, `up`, `down`, `left`, `right`.

Menus, and anything else System Events can do, go through AppleScript on stdin:

```bash
testvm script <<'EOF'
tell application "System Events" to tell process "Skillscout"
  click menu item "Settings…" of menu "Skillscout" of menu bar 1
end tell
EOF
```

`testvm logs Skillscout` prints what the app wrote to stdout and stderr since launch, so `print()` debugging works. `testvm quit Skillscout` quits it. Run `testvm help` for the full list, or `testvm capabilities` for the agent manifest.

## Other agents

Other agents can be in the VM at the same time as you. Their clicks and keys go to whatever app is in front, just like yours, so a screenshot can show another agent's app. Bring your app to the front with `testvm shot <Name>` before you click. When you move on to testing something else, add another `testvm note`.

## Data and settings

The VM is a clean Mac with one user, `admin`. It has none of the user's files, so an app that reads their data starts empty. Copy in what the test needs, usually a small sample:

```bash
testvm push ~/Documents/sample-notes/ Documents/sample-notes/
testvm run 'defaults write com.flaviocopes.skillscout sortSkillsByUse -bool true'
```

Paths on the VM side are relative to its home folder. Never copy secrets, tokens or keychains into the VM.

The VM has no Apple ID, so iCloud, Sign in with Apple and similar features don't work there. Ask the user before testing those on this Mac.

## When the VM misbehaves

- `testvm status` says whether it's running. `testvm stop` then `testvm start` restarts it. The boot log is `~/Library/Logs/testvm.log`.
- If a prompt about `com.apple.sshd-session` bypassing the private window picker covers the screen, approve it with `testvm script <<< 'tell application "System Events" to click button "Allow" of window 1 of process "UserNotificationCenter"'`.
- `testvm ssh` opens a shell in the VM. The `admin` password is `admin`, `sudo` needs no password, and Homebrew is installed.
- If screenshots come back black or clicks do nothing after a macOS update in the VM, the permission grants are gone. Recreating the VM fixes it: `tart delete mac-test` then `testvm setup`. Setup downloads about 25 GB if Tart doesn't have the image cached, so ask the user first.

`testvm` can also drive a real Mac, like a Mac mini, when `TEST_HOST` is set in `~/.config/testvm/config`. Every command works the same way.
