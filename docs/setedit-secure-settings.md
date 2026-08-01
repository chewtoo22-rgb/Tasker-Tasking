# SetEdit + Tasker: elevated Settings-DB permissions

Goal: get both **SetEdit** (`by4a.setedit22`) and **Tasker** (`net.dinglisch.android.taskerm`)
permission to read/write the `Settings.Secure` and `Settings.Global` tables, so Tasker
Tasks can change system settings that are normally locked down (no root required), and
SetEdit can be used as a GUI to inspect/edit the same tables by hand.

This is a manual, one-time step you run yourself from a computer — there's no
in-repo or in-app way to grant this permission, and this session has no access to
your phone. Everything below is the exact recipe to run.

## Why this is needed

Since Android 4.2, `WRITE_SECURE_SETTINGS` is a "signature|privileged" permission —
it's not requestable through the normal in-app permission prompt. Without root, the
only way to grant it is via `adb shell pm grant`, which works because the shell user
has elevated privileges to grant that specific permission on your behalf.

Once an app holds `WRITE_SECURE_SETTINGS`:
- **Tasker** unlocks/reliably runs actions that touch secure or global settings
  (e.g. things like immersive mode, some display/lock-screen toggles), and lets its
  **Run Shell** action call the `settings` binary directly (`settings put secure|global ...`)
  and have it actually take effect, since the shell subprocess inherits Tasker's own
  permissions.
- **SetEdit** can write to the Secure/Global tables in its editor instead of only
  reading them.

SetEdit does **not** expose a Tasker plugin / intent API — it's a standalone GUI tool.
So the "integration" here is: use SetEdit to browse and find the exact table/key/value
you need, then have Tasker apply it at runtime via `Run Shell` + `settings put`. Treat
SetEdit as the inspector, and Tasker's shell action as the thing that actually runs
during an automation.

## Prerequisites

- A computer with [`adb`](https://developer.android.com/tools/adb) installed (Android
  Platform Tools).
- USB debugging enabled on the phone: **Settings → About phone → tap Build number x7**
  to unlock Developer Options, then **Settings → Developer options → USB debugging (on)**.
- Both apps installed from Play Store first:
  - Tasker: `net.dinglisch.android.taskerm`
  - SetEdit: `by4a.setedit22`
- Phone connected via USB (or `adb` over Wi-Fi), with the RSA debugging prompt accepted.

## Steps

1. Confirm the device is seen:
   ```
   adb devices
   ```
   It should list your device as `device` (not `unauthorized`).

2. Grant the permission to both apps:
   ```
   adb shell pm grant net.dinglisch.android.taskerm android.permission.WRITE_SECURE_SETTINGS
   adb shell pm grant by4a.setedit22 android.permission.WRITE_SECURE_SETTINGS
   ```
   No output means success. `pm grant` will error out loudly if a package name is wrong
   or the permission doesn't apply.

3. Verify the grant:
   ```
   adb shell dumpsys package net.dinglisch.android.taskerm | grep WRITE_SECURE_SETTINGS
   adb shell dumpsys package by4a.setedit22 | grep WRITE_SECURE_SETTINGS
   ```
   You want to see `granted=true` next to the permission for each package.

4. In SetEdit, open **Secure** or **Global** table — it should now allow edits instead
   of showing them read-only.

5. In Tasker, add a **Run Shell** action to a Task and test with a harmless read first,
   e.g.:
   ```
   settings get secure immersive_mode_confirmations
   ```
   then a real write once you know the key/value you want, e.g.:
   ```
   settings put secure immersive_mode_confirmations 1
   ```

## No-root, no-PC method: Shizuku + Tasker's Run Shell

If you have [Shizuku](https://shizuku.rikka.app/) installed, you can skip the PC/ADB
cable entirely. Tasker 6.6+ added a **"Use Shizuku"** toggle directly on the **Run
Shell** action, so Tasker can execute privileged commands (like `pm grant`) through
the Shizuku service, all from a single on-device Task.

### 1. Start the Shizuku service (one-time per boot, on-device)

On Android 11+, this doesn't need a computer:

1. **Settings → Developer options → Wireless debugging** → turn it on.
2. Tap **Pair device with pairing code**, note the code/IP/port shown.
3. Open **Shizuku** → **Start via Wireless debugging** → enter the pairing info.
4. Shizuku's notification should show "Shizuku is running."

Note: on most non-rooted devices this has to be redone after every reboot, since
Wireless debugging's pairing doesn't auto-persist. (aShell/iAdb aren't needed for this
part — they're their own terminal apps that can also talk to Shizuku, but Tasker talks
to Shizuku directly.)

### 2. Build the Tasker Task

In the Tasker app:

1. **Tasks tab → `+`** → name it e.g. `Elevate SetEdit`.
2. Add action **Code → Run Shell**.
   - Command: `pm grant by4a.setedit22 android.permission.WRITE_SECURE_SETTINGS`
   - Enable the **Use Shizuku** toggle on this action.
3. (Optional) Add a second **Run Shell** action the same way for Tasker's own package,
   if you want Tasker itself to also hold the permission directly:
   `pm grant net.dinglisch.android.taskerm android.permission.WRITE_SECURE_SETTINGS`
4. Add action **App → Launch App** → select **SetEdit (Settings Database Editor)**.
5. Run the Task once manually and confirm:
   - Shizuku shows no error/denial popup.
   - SetEdit opens.
   - Verify the grant actually stuck, e.g. via aShell or iAdb: 
     `dumpsys package by4a.setedit22 | grep WRITE_SECURE_SETTINGS` should show `granted=true`.

### 3. Save it into this repo

Once the Task works on-device: long-press it in Tasker → **Export** → save the
`.tsk.xml`, then drop it into `tasks/` here (see the main README's export workflow)
so it's version-controlled.

## Notes / gotchas

- This grant does **not survive an uninstall/reinstall** of either app — you'll need to
  re-run step 2 if you reinstall.
- A factory reset or new device requires redoing this whole setup.
- Some OEM Android builds (especially heavily modified ones) restrict `pm grant` further;
  if step 2 fails with a `SecurityException`, that vendor's build may not allow it without root.
- Keep USB debugging off when not actively using `adb`, since it's a meaningful attack
  surface if the device is ever in someone else's hands while unlocked.
- Shizuku's service typically doesn't survive a reboot on non-rooted devices, so the
  `Elevate SetEdit` Task will fail until you restart Shizuku (step 1) again. You can
  automate *starting* Shizuku with a Tasker profile if it bothers you enough, but that's
  a separate task from this one.
