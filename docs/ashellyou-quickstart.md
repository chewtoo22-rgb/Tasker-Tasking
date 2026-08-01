# The actual easy way: grant permissions directly in aShellYou

You already have this working — you just didn't need the extra steps. **Skip
Termux, skip ADB wireless pairing, skip Tasker's Run Shell action entirely for
this part.** aShellYou (`in.hridayan.ashell`) already gives you a privileged
shell backed by Shizuku. Type commands straight into it, no prefix needed.

## 1. Grant both permissions (run these two lines in aShellYou)

```
pm grant by4a.setedit22 android.permission.WRITE_SECURE_SETTINGS
pm grant net.dinglisch.android.taskerm android.permission.WRITE_SECURE_SETTINGS
```

No output = success. `pm grant` is silent when it works. (Do **not** type `adb
shell` in front of these — aShellYou isn't `adb`, and there's no `adb` binary
installed on the phone itself, which is what caused the `adb: inaccessible or
not found` error you saw.)

## 2. Verify it actually stuck

```
dumpsys package by4a.setedit22 | grep WRITE_SECURE_SETTINGS
dumpsys package net.dinglisch.android.taskerm | grep WRITE_SECURE_SETTINGS
```

Look for `granted=true` in the output for each.

## 3. Ignore the "user 150" errors

Any line like:
```
Error: java.lang.SecurityException: Shell does not have permission to access user 150
```
is Android failing to enumerate a **different Android user/profile** on your
device (most likely Samsung's Secure Folder, which runs apps under a separate
hidden user ID). It's unrelated to Tasker or SetEdit, which both live under
your normal profile — that's why the package list command still printed both
`net.dinglisch.android.taskerm` and `by4a.setedit22` correctly despite the
error at the end. Safe to ignore for this purpose.

## 4. Now the Tasker Task can be trivial

Since the permission is granted once, permanently (until an app is
uninstalled/reinstalled), Tasker doesn't need to grant anything itself anymore.
Delete the `Run Shell` / Shizuku action from your `Elevate SetEdit` Task — it's
now dead weight and was the actual thing freezing before. The whole Task is
just:

1. Action: **App → Launch App** → SetEdit

That's it — one action, nothing that can hang. Do the permission grant once via
aShellYou as above (redo it only if you reinstall SetEdit or Tasker), and let
the Tasker Task's only job be opening the app.

## If you ever need to redo the grant

Just re-run the two `pm grant` lines from step 1 in aShellYou again — no need
to re-pair anything, since aShellYou keeps talking to Shizuku the same way
each time.
