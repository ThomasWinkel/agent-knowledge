---
title: Visio automation scripts - watchdog against hidden modal dialogs
description: Read when an agent runs Visio COM or UI-test scripts - a modal dialog blocks the COM call silently; detect it from a second process and abort; dump a hung Visio with cdb first.
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

## Dump a hung Visio before the kill

When Visio stops answering COM with no dialog (stall), a full dump taken before the kill shows where its UI thread waits. WinDbg from winget (`winget install Microsoft.WinDbg`; ProcDump is not in winget) ships `cdb.exe` in its app package. Attach non-invasively (`-pv`: no debug events, threads only suspended while attached), write the dump, detach (`qd`); no symbols needed for writing. Commands from a file avoid quoting trouble with paths in `-c`:

```powershell
$cdb = Join-Path (Get-AppxPackage Microsoft.WinDbg).InstallLocation 'amd64\cdb.exe'
$cmd = Join-Path $env:TEMP "cdb-$procId.txt"
".dump /ma /o `"$dumpPath`"`nqd" | Out-File $cmd -Encoding ascii
$p = Start-Process $cdb -ArgumentList "-pv -p $procId -cf `"$cmd`"" -RedirectStandardOutput "$dumpPath.cdb.txt" -WindowStyle Hidden -PassThru
if (-not $p.WaitForExit(180000)) { Stop-Process -Id $p.Id -Force }   # then kill Visio as usual
```

Also dump the script's own PowerShell process: it shows the COM call that waits (on a pipeline thread, not thread 0). A Visio still running ~10 s after the script ended (Quit did not finish) is worth a dump too.

Read it later (first read downloads symbols, ~1 min; then seconds):

```powershell
& $cdb -z <dump> -y 'srv*C:\symbols*https://msdl.microsoft.com/download/symbols' -c '~*k 40; .loadby sos clr; ~*e !clrstack; !syncblk; !threads; q'
```

`.loadby sos clr` = SOS of .NET Framework 4 (VSTO add-ins); `lm m <AddInAssembly>*` lists the add-in modules. Visio has no public symbols: its frames show export names with large offsets (`VISLIB!MLFlushNotifies+0x2a80`), so read the boundaries to OLE/CLR/add-in frames.

Measured 2026-11, Visio 2019 C2R x64 build 20527 with a VSTO add-in, invisible instance: dump 846-858 MB in 2-6 s; PowerShell 5.1 237 MB in 1 s (once 64 s, cause unknown).
