# claude-skills: reference

Detail moved out of the [README](../README.md): what each skill does, the repository layout, installing and changing
a skill.

## The skills

| Plugin | Invoke as | What it does | Project notes it reads |
|---|---|---|---|
| [`ui-lab`](../plugins/ui-lab/skills/ui-lab/SKILL.md) | `/ui-lab:ui-lab` | UX/UI suggestions as tappable mocks in the app's UI lab: mock, close the round, promote | `.claude/ui-lab.md` |
| [`android-emulator`](../plugins/android-emulator/skills/android-emulator/SKILL.md) | `/android-emulator:android-emulator` | Start and stop the project's emulator, pick phone or emulator, install, open, screenshot | `.claude/android-emulator.md` |
| [`qnn-npu-compile`](../plugins/qnn-npu-compile/skills/qnn-npu-compile/SKILL.md) | `/qnn-npu-compile:qnn-npu-compile` | Compile ONNX models for Qualcomm NPUs (QNN HTP) and run them on Android | none |

Claude Code also picks a skill up on its own when a request matches the skill's description.

### ui-lab

A three-phase process for UX changes, so that a design is tried on a device before any app code changes:

1. **Mock.** The agent writes two or three distinct options for a screen, each in its own folder of lab-only code,
   on a `ux/<screen>` branch. Each option lists what it fakes. The owner taps through them on a phone.
2. **Close the round.** The round is tagged in git (`ux/<screen>/<date>`), written up in `docs/ux-history/`, and
   the options that were not chosen are deleted. Any option can be revived later from its tag.
3. **Promote.** After the owner approves a plan, the chosen option's screen file moves into the app, the logic it
   faked is built with tests, and a release build confirms nothing references lab code.

### android-emulator

How an agent gets an Android app running and looks at it: prefer the owner's phone over the emulator, always address
a device by serial, build only the device's ABI, install without wiping data, open the app so the new build really
runs, then take screenshots, tap through a flow and read logs. It starts the emulator only while needed and stops it
afterwards. The bundled
[scripts/emulator.sh](../plugins/android-emulator/skills/android-emulator/scripts/emulator.sh) has `up`, `status` and
`down`; it is written for macOS paths by default and reads `ANDROID_HOME`.

It includes safety rules for a real phone: never uninstall or clear the owner's app without asking, change the
locale only on the emulator, and put back any setting it changed.

### qnn-npu-compile

A procedure for running a model on a Snapdragon phone's NPU (Hexagon HTP through the QNN / QAIRT SDK), following how
sherpa-onnx builds its published NPU models:

- decide first whether the NPU is worth it: measure on the phone, static shapes only, one chip family per build,
  always keep a CPU fallback;
- set up the tools (the SDK runs on Linux x86-64, or in Docker on a Mac);
- re-export the ONNX from the original checkpoint with fixed shapes and no int8 quantization;
- convert, build and compile a context binary per graph and per chip;
- load it on the phone with sherpa-onnx or ONNX Runtime's QNN execution provider;
- verify: same output as the CPU, speed, and power over a fixed task;
- a list of known pitfalls (dynamic shapes, SDK version mismatch, silent CPU fallback, licenses).

## Repository layout

```
.claude-plugin/marketplace.json                 the marketplace: name chencohen-skills, three plugins
plugins/<name>/.claude-plugin/plugin.json       a plugin's name, version and description
plugins/<name>/skills/<name>/SKILL.md           the skill: front matter (name, description) and the instructions
plugins/android-emulator/skills/android-emulator/scripts/emulator.sh   the one bundled script
```

## How to install and use

Add to a project's `.claude/settings.json` and commit it, listing only the plugins that project uses:

```json
{
  "extraKnownMarketplaces": {
    "chencohen-skills": {
      "source": { "source": "github", "repo": "chencohensolutions/claude-skills" }
    }
  },
  "enabledPlugins": {
    "ui-lab@chencohen-skills": true,
    "android-emulator@chencohen-skills": true
  }
}
```

Claude Code offers to install them the next time the project is opened (after the folder is trusted).
Or from a terminal:

```bash
claude plugin marketplace add chencohensolutions/claude-skills
claude plugin install ui-lab@chencohen-skills --scope project
```

Then, for `ui-lab` and `android-emulator`, add the project's notes file (`.claude/ui-lab.md`,
`.claude/android-emulator.md`). Each skill lists what its notes file should contain, and tells the agent to ask and
write the notes when the file is missing.

## Add or change a skill

- New skill: `plugins/<name>/.claude-plugin/plugin.json` and `plugins/<name>/skills/<name>/SKILL.md`, plus its
  entry in `.claude-plugin/marketplace.json`.
- Changing one: bump `version` in its `plugin.json`, push, then `claude plugin marketplace update chencohen-skills`
  on each machine.
