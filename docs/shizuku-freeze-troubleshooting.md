# Why the "Elevate SetEdit" Task freezes / does nothing

If the Task from `setedit-secure-settings.md` hangs or silently does nothing, it's
almost always one of two things:

1. **Shizuku isn't actually running** (its wireless-debugging session died — this is
   normal, it usually doesn't survive a reboot or Wi-Fi drop), so the Run Shell action
   sits waiting on a service that isn't there.
2. **Shizuku is running, but this is the *first* time Tasker has asked it for
   permission.** Shizuku pops up its own "Allow Tasker to use Shizuku?" dialog the
   first time — if the Task ran in the background (triggered by a Profile, or you
   switched apps right after tapping Run), that popup can appear and just sit there
   unanswered, or get killed by Android's background-activity restrictions. Tasker's
   Run Shell action has **no timeout by default**, so it waits forever = "freezes."

The fix below does two things: makes the Task **fail loud with a message** instead of
hanging forever, and walks through testing each piece **in isolation** so you can see
exactly which step is broken.

## Step 0: rebuild the Task to fail loud instead of hanging

Edit your `Elevate SetEdit` Task (or rebuild it) with these settings — the extra
actions are all diagnostic, you can trim them later once it's working:

1. **Run Shell**
   - Command: `pm grant by4a.setedit22 android.permission.WRITE_SECURE_SETTINGS`
   - **Use Shizuku**: on
   - **Timeout (Seconds)**: `10` — this is the important part. Without it, a dead
     Shizuku connection hangs indefinitely instead of erroring out.
   - **Store Result In**: `%grant_result`
2. **Flash** (or **Notify**) — text: `Grant result: %grant_result %err`
   - This shows you immediately whether the shell command ran, timed out, or errored,
     instead of you having to guess why nothing happened.
3. **Launch App** → SetEdit
   - Only add this after step 1–2 report success at least once. Keep it separate while
     debugging so a hang in the grant step doesn't also hide whether SetEdit launches
     fine on its own.

With a 10s timeout, "freezes forever" becomes "fails after 10 seconds and tells you
the result" — much easier to work from.

## Step 1: confirm Shizuku itself is alive, outside of Tasker

Open the **Shizuku** app directly:
- Does it say **"Shizuku is running"**? If it says **"not running"** / "start via
  wireless debugging" again, that's your whole problem — the Task was never going to
  work no matter what's in it, because the service it depends on isn't up. Re-pair
  (Settings → Developer options → Wireless debugging → pair with code → back into
  Shizuku → Start).

If you don't want to re-pair by hand every reboot, that's a separate, solvable
annoyance — see "Auto-starting Shizuku" below.

## Step 2: test a trivial Shizuku command from Tasker, alone

Make a throwaway Task, just:
- **Run Shell**: `echo hi`, **Use Shizuku** on, **Timeout**: `10`, **Store Result In**: `%t`
- **Flash**: `%t %err`

Run it manually while watching the screen. Three outcomes:
- Flash shows `hi` → Shizuku↔Tasker link works fine, the problem is specific to the
  `pm grant` command or SetEdit — go to Step 3.
- A **Shizuku permission popup appears** asking to allow Tasker → tap **Allow**, then
  re-run the Task. This was the one-time approval that was getting missed.
- Flash never appears / Task still hangs past 10s → Shizuku isn't actually reachable;
  go back to Step 1, the service isn't really up despite what its notification says.

## Step 3: test `pm grant` outside Tasker, via aShell or iAdb

Since you have aShell/iAdb, run this directly in one of them (they can use Shizuku as
their backend too — check that app's own settings for a Shizuku toggle):
```
pm grant by4a.setedit22 android.permission.WRITE_SECURE_SETTINGS
```
- Works here but not from Tasker → the problem is in the Tasker action config
  specifically (double check "Use Shizuku" is actually toggled on for that action, not
  just typed into the command).
- Fails here too → it's not a Tasker/Shizuku problem at all. Common causes: package
  name typo (confirm with `pm list packages | grep setedit`), or Shizuku's granted
  permission level doesn't cover `pm grant` on your OEM's build.

## Step 4: confirm the grant actually stuck

```
dumpsys package by4a.setedit22 | grep WRITE_SECURE_SETTINGS
```
Look for `granted=true`. If the command reported success but this still shows
`granted=false`, something reset it (reinstall, OEM permission auto-revoke for
unused apps — check Settings → Apps → SetEdit → "remove permissions if unused" and
turn that off).

## Auto-starting Shizuku (optional, once the above works)

Manually re-pairing Shizuku after every reboot gets old. Options, roughly easiest to
hardest:
- Shizuku itself can auto-start on boot **only on rooted devices** — not applicable here.
- On non-rooted devices, some people wire a Tasker profile on **Display On** /
  **Wi-Fi Connected** that opens the Shizuku app to its "Start" screen as a nudge, but
  you still have to tap Start yourself — Android doesn't allow silently
  auto-approving a debugging connection, that's a deliberate security boundary, not
  something a Task can script around.
- If this friction is a dealbreaker, rooting is the only way to get a truly
  silent, boot-persistent privileged shell for Tasker.
