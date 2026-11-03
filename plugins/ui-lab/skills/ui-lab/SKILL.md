---
name: ui-lab
description: Build UX/UI suggestions as tappable mocks in the app's UI lab, record each round in docs/ux-history, and promote the chosen option into the app once the owner approves a plan. Use whenever the owner asks to suggest, mock up, compare or try UX/UI designs for a screen, or to bring back an earlier suggestion.
---

# UI lab: UX suggestions

A UX suggestion goes through three phases:

1. **Mock** — only lab code changes, on the platform the project's lab notes say. The owner taps through the options.
2. **Close the round** — the round is tagged and written up in `docs/ux-history/`, and its lab files are deleted.
3. **Promote** — after the owner picks an option and approves a plan, it becomes the real screen, with its shared
   logic and the other platform when the app has one.

## The project's lab notes

This skill is the process; each project's specifics live in its **`.claude/ui-lab.md`**. Read it before anything
else. It gives:

- **Platform**: which lab app to use on this machine (`uname -s`), its id, where lab code goes and the command that
  opens a screen;
- **Option folders**: the folder and package per platform, and where strings go;
- **Files**: the theme or colors an option's screen uses, and where its state moves on promotion;
- **Sample data**: what the options build their data from;
- **Fakes and edge cases**: what made-up data can't show in this app, and the cases to check;
- **Screenshots**: how to take one per platform;
- **Promote**: target paths, the other platform's handling, docs to update and the release build commands;
- how the lab itself works (a doc link).

The notes win where they differ from this skill. If the project has no `.claude/ui-lab.md`, or it lacks something
below needs, ask the owner rather than guessing, and add the answer to the notes.

## Phase 1: mock

Work on a `ux/<screen>` branch from the current one, so another machine can pull the round. If the tree has
uncommitted changes that aren't yours, ask before switching.

### One folder per option

Each option lives in its own folder (the notes say where), so it can be deleted, revived or promoted alone. It sits
in the screens' package, so promoting it is a file move.

| File | Holds | On promotion |
|---|---|---|
| `<Screen>Option<B>` | the screen, `<Screen>Screen<B>(state, actions)`: production quality (the app's strings and theme, per the notes) and free of lab types | moves into the app |
| `<Screen>Option<B>State` | state the app's logic doesn't produce yet, shaped as it would produce it | moves where the notes say |
| `<Screen>Option<B>Lab` | its sample data, fake behavior, do-nothing actions, and its `LabOption` value with what it fakes above it | deleted |
| the option's strings, when the app uses string resources (names and location per the notes) | the option's new text | moves to the app's strings |

- Besides the folder (and strings files where resources can't sit in it), an option adds one line: its entry in its
  screen's list in `LabScreens` (`<screen>OptionB,`).
- Everything in the folder is `private` except the screen, its state and the `LabOption` value, whose names carry
  the option, so options never clash.
- An option that needs a changed version of an existing app component uses its own copy, in its folder.
- Sample data comes from the sources the notes name, as they are, plus what the option adds in its `Lab` file. Don't
  invent text where the notes say the data is real.

### Steps

1. Open the screen in the lab and look at `current`, the screen as the app has it now.
2. Write 2–3 distinct directions as options `a`, `b`, `c` after `current`, each with a one-line note on what it
   changes.
3. Above each `LabOption` value, list what the option fakes: data the app's logic lacks, behavior that isn't built,
   and what made-up data can't show (the notes list this app's usual ones).
4. Check the edge cases the notes list, plus whatever is unusual for the screen. A case that changes the design
   becomes its own option (`a-rtl`, `a-dark`).
5. Build, open each option and look at a screenshot (commands in the notes; the android-emulator skill covers
   Android devices). An all-black screenshot from a phone means it is locked: ask the owner to unlock it.
6. Commit on the branch and tell the owner the option ids (`<screen> <option>`), what each fakes, and the device.

When the owner asks to change an option, the change becomes a new option in its own folder (`b2` beside `b`).
Options the owner rules out stay until the round closes, so the round's tag has them all.

## Phase 2: close the round

When the owner has picked an option or turned them all down:

1. Tag the round's last commit, while every option is still in it, and push the tag:
   `git tag ux/<screen>/<yyyy-mm-dd> && git push origin ux/<screen>/<yyyy-mm-dd>`. The tag keeps the code reachable
   after the branch is gone; a second round the same day gets `-2`.
2. Write `docs/ux-history/<screen>/<yyyy-mm-dd>-<topic>.md` in the format below, and link it from wherever the
   notes say (a screen's doc, for instance).
3. Delete the other options' folders, strings files and `LabScreens` lines; the chosen option stays until phase 3.
4. Commit. With every option turned down, the round ends here: merge the branch (the history file is its only
   lasting change) or delete it, as the owner prefers.

```markdown
# <Screen> — <topic> (<yyyy-mm-dd>, <platform>)

Tag: `ux/<screen>/<yyyy-mm-dd>`

## Options
- **a — <name>**: <what it changes>. *Picked*, promoted in <commit>.
- **b — <name>**: <what it changes>. *Turned down*: <the owner's reason, when given>.

## What they faked
- b: <what the mock showed that the app can't do yet>.

## Reviving an option
    git checkout ux/<screen>/<yyyy-mm-dd> -- <option folder> <strings files>
Then add `<screen>Option<B>,` to `LabScreens`.
Built against <components and sample data it used> as of the tag. Compile errors after reviving are in the
option's copies and sample data; fix them there.
```

### Reviving

When the owner asks for an earlier option, follow its history file: check out the paths from the tag onto a
`ux/<screen>` branch, add the `LabScreens` line, fix what no longer compiles inside the option's folder, and open it.
It is then a live option of a new round.

## Phase 3: promote

With the round closed on a picked option, enter plan mode (`EnterPlanMode`) and present the plan with
`ExitPlanMode`; the owner approves it with its button. The plan covers:

- the files that move and the screen they replace;
- the logic work behind each faked item: the proposed state, view model or controller changes, and their tests;
- the other platform, when the app has one, as the notes say;
- the docs the notes say to update.

After approval, on the same branch:

1. `git mv` the option's screen file into the app (target per the notes), replacing the old screen, and give the
   function or view the screen's name.
2. Move its strings into the app's, without the option's prefix or table.
3. Move its state where the notes say, have the app's logic produce it with tests, and connect the actions.
4. Fold its copied components back into the originals, and delete the rest of its folder and its `LabScreens` line;
   point the screen's `current` in `LabScreens` at the promoted screen.
5. Build release with the notes' commands, where any reference to lab code fails to compile.
6. Do the plan's other-platform and docs parts, and put the promoting commit in the round's history file.

Merging the branch is the owner's call.
