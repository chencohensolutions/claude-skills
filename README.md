# claude-skills

Shared [Claude Code](https://claude.com/claude-code) skills, packaged as a plugin marketplace. Each skill is its own
plugin, so every project enables only the skills it uses.

| Plugin | Skill | What it does | Project notes it reads |
|---|---|---|---|
| `ui-lab` | `/ui-lab:ui-lab` | UX/UI suggestions as tappable mocks in the app's UI lab: mock, close the round, promote | `.claude/ui-lab.md` |
| `android-emulator` | `/android-emulator:android-emulator` | Start and stop the project's emulator, pick phone or emulator, install, open, screenshot | `.claude/android-emulator.md` |
| `qnn-npu-compile` | `/qnn-npu-compile:qnn-npu-compile` | Compile ONNX models for Qualcomm NPUs (QNN HTP) | — |

The skills hold the process; each project keeps its own paths, ids and commands in the notes file named above.

## Enable skills in a project

Add to the project's `.claude/settings.json` and commit it, listing only the plugins that project uses:

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

## Add or change a skill

- New skill: `plugins/<name>/.claude-plugin/plugin.json` and `plugins/<name>/skills/<name>/SKILL.md`, plus its
  entry in `.claude-plugin/marketplace.json`.
- Changing one: bump `version` in its `plugin.json`, push, then `claude plugin marketplace update chencohen-skills`
  on each machine.
- Check before pushing: `claude plugin validate .`
