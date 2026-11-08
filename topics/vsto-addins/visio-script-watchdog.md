---
title: Visio automation scripts - watchdog against hidden modal dialogs
description: Read when an agent runs Visio COM or UI-test scripts - a modal dialog blocks the COM call silently; detect it from a second process within seconds and abort.
---

# Watchdog for Visio automation scripts

A modal dialog in Visio (Office `NUIDialog`, Win32 `#32770`, add-in message box) blocks the running COM call; the script waits without output until a tool timeout, often many minutes. `Application.AlertResponse` answers only some alerts (not dialogs opened by `DoCmd`, see [ribbon-ui-test](ribbon-ui-test.md)). Run the script in its own process and watch it from outside.

Checks every 2 s, for Visio processes started after the watchdog:

- **Main window disabled**: a visible top-level window of class `VISIOA` with `IsWindowEnabled == false` = some modal dialog is open (catches every dialog type).
- **Dialog window**: visible top-level window of class `#32770` or `NUIDialog`; report its title (`GetWindowText`) and child texts (`EnumChildWindows` + `GetWindowText`). Re-check after ~3 s so short progress boxes do not count.
- **Stall**: the script's result file has not grown for N s (script logs line by line).
- **Timeout** in all.

On abort: kill the script process and those Visio processes, append the reason to the result file.

```csharp
// Add-Type in PowerShell; pids = Visio processes started by the script
EnumWindows((h, l) => {
    GetWindowThreadProcessId(h, out uint pid);
    if (!pids.Contains(pid) || !IsWindowVisible(h)) return true;
    string cls = ClassName(h);
    if (cls == "VISIOA" && !IsWindowEnabled(h)) found.Add("main window disabled: " + Text(h));
    if (cls == "#32770" || cls == "NUIDialog") found.Add(cls + " | " + Text(h) + " | " + ChildTexts(h));
    return true;
}, IntPtr.Zero);
```

Verified 2026-11, Visio 2019 C2R x64: `DoCmd(2165)` (Page Setup, `#32770`) blocked the script; the watchdog reported both the `#32770` window with its texts and the disabled `VISIOA` window 7 s after start, then killed Visio.
