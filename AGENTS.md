# testvm

A bash script that runs Mac app builds in a headless macOS VM under Tart, so coding agents can open, screenshot and click through them over SSH without taking over the user's screen. No dependencies beyond Tart on the Mac, and cliclick in the VM.

- `testvm`: the whole tool. Settings and their defaults come first, then the commands, then the activity log every command writes. The `capabilities` manifest, `VERSION` and the changelog live at the top.
- `skills/mac-test-vm/SKILL.md`: the agent skill that teaches agents to use `testvm`. Users install it with `npx skills add flaviocopes/testvm`.
- `README.md`: install, commands, settings, testing on another Mac, licenses.

## Check a change

```bash
bash -n testvm                                       # syntax
./testvm --version
./testvm capabilities --json | python3 -m json.tool  # a stray quote breaks the JSON silently
./testvm status
./testvm shot                                        # whole VM screen, into /tmp/testvm
```

Run the changed command against the real VM from the repo, as `./testvm`. Other agents can be using the VM, so check `tail ~/Library/Logs/testvm/activity.jsonl` first. If someone used it in the last few minutes, stick to commands that don't click, type or press keys.

## Rules

- Never edit an installed copy of `testvm` in place. Write a new file and `mv` it over the old one: bash reads a script while it runs, and agents run `testvm` all the time.
- The log in `~/Library/Logs/testvm` is a contract with [VM Peek](https://github.com/flaviocopes/vm-peek), which parses it in `Sources/MonitorCore/ActivityLog.swift` and `Run.swift`. Keep old lines readable: new fields are optional, and existing fields keep their meaning. VM Peek also reads `TEST_HOST` and `TEST_USER` from the config and uses the same SSH options and `ControlPath`.
- Logging must never break a test, so every write to the log ignores its errors.
- On every release, bump `VERSION` and add the changelog entry in both `capabilities_json` and `print_capabilities`, in the same commit. Versions follow semver.
- When a command changes, update `usage`, the command list in `README.md` and the skill together.
- A release is a `vX.Y.Z` tag with the `testvm` script attached as a release asset, because the README installs from `releases/latest/download/testvm`.
- No GitHub Actions or other CI.
