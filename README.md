# claude-skills

Three reusable [Claude Code](https://claude.com/claude-code) skills, packaged as a plugin marketplace: a UX mock-up
workflow, Android device and emulator handling, and compiling ONNX models for Qualcomm NPUs.

## At a glance

- **Problem:** an AI coding agent repeats the same multi-step procedures in every project, and gets them slightly
  wrong each time unless the procedure is written down once.
- **What it is:** each procedure is a skill, a Markdown instruction file (`SKILL.md`) that Claude Code loads when a
  task matches. Each skill is its own plugin, so a project enables only the ones it uses.
- **Design rule:** the skill holds the process. Each project keeps its own paths, ids and commands in a small notes
  file (`.claude/ui-lab.md`, `.claude/android-emulator.md`), so the same skill works across projects without edits.
- **Contents:** 3 plugins, 3 skills (306 lines of instructions) and one Bash helper script (66 lines), all at
  version 0.1.0.
- **Tests:** none. The content is instructions. Manifests are checked with `claude plugin validate .`.

**Stack:** Claude Code, agent skills, Claude Code plugins and plugin marketplace, AI coding agents, agentic
workflows, prompt engineering, Bash, Android (adb, emulator, Gradle), ONNX, Qualcomm QNN / QAIRT, Hexagon NPU,
sherpa-onnx, ONNX Runtime, model quantization.

## My role

I (Hen Cohen) wrote these skills from procedures I worked out with AI coding agents in my own projects. I then
generalized them so the project-specific parts live in each project's notes file. They are instructions for an
agent, so the engineering is in the wording: what the agent must check, what it must ask the owner before doing, and
where a step usually goes wrong.

## The skills

| Plugin | What it does |
|---|---|
| [`ui-lab`](plugins/ui-lab/skills/ui-lab/SKILL.md) | UX/UI suggestions as tappable mocks in the app's UI lab: mock, close the round, promote |
| [`android-emulator`](plugins/android-emulator/skills/android-emulator/SKILL.md) | Start and stop the project's emulator, pick phone or emulator, install, open, screenshot |
| [`qnn-npu-compile`](plugins/qnn-npu-compile/skills/qnn-npu-compile/SKILL.md) | Compile ONNX models for Qualcomm NPUs (QNN HTP) and run them on Android |

## Quick start

```bash
claude plugin marketplace add chencohensolutions/claude-skills
claude plugin install ui-lab@chencohen-skills --scope project
```

Then invoke a skill as `/ui-lab:ui-lab`, or let Claude Code pick it up when a request matches. For `ui-lab` and
`android-emulator`, add the project's notes file. The skill tells the agent to ask and write it when it is missing.
The `settings.json` way to install is in [docs/REFERENCE.md](docs/REFERENCE.md).

## Tests

```bash
claude plugin validate .    # checks the manifests; there are no automated tests
```

## Status

- First published Nov 3, 2026. All three plugins are at 0.1.0.
- The Android skill and its script assume a macOS host by default.
- No license file is included yet.

## More docs

- [docs/REFERENCE.md](docs/REFERENCE.md): what each skill makes the agent do, repository layout, installing through
  `settings.json`, adding or changing a skill.
