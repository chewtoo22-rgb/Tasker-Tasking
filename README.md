# Tasker Tasking

Version-controlled backups and history for [Tasker](https://play.google.com/store/apps/details?id=net.dinglisch.android.taskerm) — the Android automation app. Tasker doesn't talk to git directly, so this repo holds the XML each piece exports to, and this README documents the workflow for getting things in and out.

## Folder structure

| Folder      | Contents                                                    |
|-------------|--------------------------------------------------------------|
| `projects/` | Whole Tasker Projects (a Project bundles Profiles + Tasks)   |
| `profiles/` | Individual Profiles (the "when" — context/event that triggers a Task) |
| `tasks/`    | Individual Tasks (the "what" — the action sequence itself)   |
| `scenes/`   | Scenes (custom UI layouts)                                    |
| `icons/`    | Any custom icons/images referenced by the above               |

Each exported item is one XML file. Suggested naming: lowercase, hyphenated, matching the name in Tasker, e.g. `projects/home-automation.prj.xml`, `tasks/send-daily-summary.tsk.xml`.

## Exporting from Tasker (phone → repo)

1. Long-press the Project / Profile / Task / Scene in the Tasker app.
2. Tap **Export**.
3. Choose a destination (e.g. a synced folder like Google Drive/Syncthing, or share directly to wherever this repo is checked out).
4. Move/rename the resulting `.xml` file into the matching folder above.
5. Commit with a message describing *why* it changed, not just "export".

## Importing into Tasker (repo → phone)

1. Get the XML file onto the device (synced folder, Play Store's "Send to" via Drive, etc.).
2. In Tasker, long-press the relevant list (Projects/Profiles/Tasks/Scenes) and choose **Add** → **Import**, or use "Import" from the overflow menu.
3. Browse to the file and select it.

## Notes

- Tasker XML embeds absolute references to other Tasks/Scenes by name, so keep names stable once other Projects depend on them.
- Prefer exporting the whole **Project** over individual Profiles/Tasks when they're tightly coupled — it keeps cross-references intact in one file.
- Treat this repo as the source of truth: if you tweak something on-device, export it back here before it's forgotten.

## Docs

- [`docs/ashellyou-quickstart.md`](docs/ashellyou-quickstart.md) — **start here.** The
  simplest working method: grant `WRITE_SECURE_SETTINGS` directly via the aShellYou app
  (no PC, no ADB pairing, no Tasker shell action needed), then Tasker's Task is just a
  single "Launch App" action.
- [`docs/setedit-secure-settings.md`](docs/setedit-secure-settings.md) — background:
  what the permission is and the PC/ADB and in-Tasker-Shizuku alternatives.
- [`docs/shizuku-freeze-troubleshooting.md`](docs/shizuku-freeze-troubleshooting.md) —
  why the in-Tasker Shizuku approach was hanging (superseded by the aShellYou method
  above, but kept for reference).
