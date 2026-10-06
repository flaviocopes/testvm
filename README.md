# testvm

`testvm` runs your Mac app builds in a headless macOS virtual machine, so coding agents can open, screenshot and click through them without taking over your screen.

I build Mac apps with Cursor, Claude Code and Codex. Every time an agent wanted to check its work, it opened the app on my screen and took the focus while I was doing something else. Now the agents build on my Mac as usual, then `testvm` copies the `.app` into a VM running under [Tart](https://tart.run) and drives it over SSH. The VM has no window, so nothing shows up on my screen.

I explain how it works, step by step, in [How to let coding agents test Mac apps in a VM](https://flaviocopes.com/mac-test-vm/).

## Install

You need an Apple silicon Mac. Install Tart first:

```sh
brew install cirruslabs/cli/tart
```

If the tap fails with your version of Homebrew, download Tart from GitHub instead:

```sh
curl -LO https://github.com/cirruslabs/tart/releases/latest/download/tart.tar.gz
tar xzf tart.tar.gz
mkdir -p ~/Applications
mv tart.app ~/Applications/
ln -s ~/Applications/tart.app/Contents/MacOS/tart /opt/homebrew/bin/tart
```

Then put `testvm` in your `PATH`:

```sh
curl -Lo /opt/homebrew/bin/testvm https://github.com/flaviocopes/testvm/releases/latest/download/testvm
chmod +x /opt/homebrew/bin/testvm
```

And create the VM:

```sh
testvm setup
```

It downloads Cirrus Labs' macOS Sequoia image, about 25 GB, and gives the VM 4 cores and 8 GB of memory. The image has SSH turned on, logs in automatically, and already grants SSH the permissions an agent needs to take screenshots and click. `setup` adds an SSH key, installs [cliclick](https://github.com/BlueM/cliclick) in the VM and approves the screen recording prompt.

The VM runs in the background, with no window and no Dock icon. When it's off, any command that needs it starts it, which takes about 15 seconds.

## Use it

Build your app as usual, then open it in the VM:

```sh
xcodebuild -scheme Skillscout -derivedDataPath build build
testvm open build/Build/Products/Debug/Skillscout.app
```

`open` copies the app to `~/Apps` in the VM, quits the copy that's running, launches the new one and waits for its window. Then take a screenshot of the window:

```sh
testvm shot Skillscout
```

It prints the path of a PNG in `/tmp/testvm`. To click something, list the window's controls first. Each one comes with its label and the point to click:

```sh
testvm ui Skillscout
```

```text
WINDOW "Skillscout"
AXPopUpButton [pop up button] = Newest first  center 855,113
AXButton "Find repeated tasks" (disabled)  center 976,113
AXTextField [search text field]  center 1171,114
```

Then click, type and press keys:

```sh
testvm click 1171 114
testvm type 'release'
testvm key cmd+a
```

Here's every command:

```text
testvm open <App.app> [args]   copy the build into the VM, launch it, wait for a window
testvm open <Name>             relaunch an app already copied into the VM
testvm shot [Name] [out.png]   screenshot the whole VM screen, or just Name's front window
testvm ui <Name>               list the app's controls with text and click coordinates
testvm click <x> <y>           click at screen coordinates (points)
testvm type <text>             type text into the frontmost app
testvm key <combo>             press keys: cmd+n, cmd+shift+s, return, esc, arrow-down
testvm script [args]           run AppleScript from stdin in the VM (System Events, menus)
testvm quit <Name>             quit the app
testvm logs <Name>             print the app's stdout/stderr since it was launched
testvm push <local> [remote]   copy files or folders into the VM (default: home folder)
testvm pull <remote> [local]   copy files out of the VM
testvm run <command>           run a shell command in the VM
testvm ssh                     open an interactive shell in the VM
testvm start | stop | status   manage the VM (other commands start it when needed)
testvm setup                   download and configure the VM from scratch
testvm note <text>             say what you're testing, shown in VM Monitor
testvm capabilities [--json]   summary and task list for agents
```

## Tell your agents to use it

Agents run `open` out of habit, so tell them not to. Add this to your Cursor rules, `~/.claude/CLAUDE.md` or `~/.codex/AGENTS.md`:

```markdown
When you test a Mac app you built, never launch, relaunch, quit, screenshot
or click through it on this Mac. Do all of that in the test VM with `testvm`.

This overrides project instructions that say to open or relaunch the app
after a change. Do those steps in the VM instead.
```

The second paragraph matters if your projects have an `AGENTS.md` that says to open the app after a change.

The details of every command live in a skill, so the rule stays short. Install it with the [skills CLI](https://skills.sh):

```sh
npx skills add flaviocopes/testvm
```

Or copy the `skills/mac-test-vm` folder into your agent's skills folder, like `~/.cursor/skills` or `~/.claude/skills`.

Codex runs commands in a sandbox without network access, and `testvm` talks to the VM over SSH. So in Codex, `testvm` has to run outside the sandbox.

## Settings

`testvm` reads `~/.config/testvm/config`, a file with one `KEY=value` per line. Every setting is optional. These are the defaults:

```sh
VM_IMAGE=ghcr.io/cirruslabs/macos-sequoia-base:latest
VM_CPUS=4
VM_MEMORY=8192
VM_DISPLAY=1512x982
```

`testvm setup` uses them when it creates the VM. I've only tested the Sequoia image, but Cirrus Labs publishes `base` images for other versions of macOS too.

To resize a VM you already have, stop it and change it with Tart:

```sh
testvm stop
tart set mac-test --cpu 6 --memory 16384
```

## Test on another Mac

`testvm` works the same way with a real Mac, like a Mac mini on your network. On that Mac:

1. Turn on Remote Login and automatic login, and keep it from sleeping.
2. Install cliclick with Homebrew.
3. In System Settings → Privacy & Security, add `/usr/libexec/sshd-keygen-wrapper` to Accessibility and Screen Recording.

If you never ran `testvm setup`, make its SSH key first:

```sh
ssh-keygen -t ed25519 -N '' -f ~/.ssh/testvm_ed25519
```

Then copy the key to the Mac mini and point `testvm` at it:

```sh
ssh-copy-id -i ~/.ssh/testvm_ed25519 flavio@mini.local
mkdir -p ~/.config/testvm
printf 'TEST_HOST=mini.local\nTEST_USER=flavio\n' >> ~/.config/testvm/config
```

The first `testvm ui` or `testvm script` shows a prompt on the Mac mini asking to let SSH control System Events. Approve it once. From then on every command goes to the Mac mini, and `testvm` checks its host key like any SSH connection. Remove the two lines to go back to the VM.

## See what your agents did

Every command goes into `~/Library/Logs/testvm/activity.jsonl`, with the agent that ran it (Cursor, Claude Code, Codex or a terminal), its chat and the folder it ran from. What the command printed goes to `runs/<id>.txt` next to it. Agents can add what they're testing with `testvm note 'Checking the new sort menu'`.

[VM Monitor](https://github.com/flaviocopes/vm-monitor) is a Mac app that reads this log. It shows the VM's screen live, which agents are using it, and every command each chat ran.

Several agents can use the VM at the same time. Their clicks and keys go to whatever app is in front, and VM Monitor warns you when two chats overlap.

## What the VM can't do

- It's a clean Mac with none of your files, so an app that reads your data starts empty. Copy in test data with `testvm push`, and never copy secrets, tokens or keychains.
- It has no Apple ID, so iCloud and Sign in with Apple don't work there.
- It uses 4 cores and 8 GB of memory while it runs, and Tart reports about 33 GB on disk.

The VM's user is `admin`, with the password `admin` and `sudo` without a password, as Cirrus Labs set up the image. Tart puts the VM on a private network that only your Mac can reach.

## Uninstall

```sh
testvm stop
launchctl bootout gui/$(id -u)/com.flaviocopes.testvm
tart delete mac-test
rm ~/Library/LaunchAgents/com.flaviocopes.testvm.plist /opt/homebrew/bin/testvm ~/.ssh/testvm_ed25519*
rm -rf ~/.config/testvm ~/Library/Logs/testvm ~/Library/Logs/testvm.log
```

## License

`testvm` is [MIT](LICENSE) licensed. It doesn't include Tart or macOS, it runs the copies you install:

- Tart uses the [Functional Source License](https://github.com/cirruslabs/tart/blob/main/LICENSE), which allows any use except a commercial product that competes with Tart.
- Apple's [macOS license](https://www.apple.com/legal/sla/) allows up to two extra copies of macOS in virtual machines on a Mac you own or control, for software development and testing, among other uses.
