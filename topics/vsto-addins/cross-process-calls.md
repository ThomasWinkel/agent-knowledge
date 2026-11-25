---
title: Calls between Office processes - COMAddIn.Object and events of another Office app
description: Read when a VSTO add-in calls the add-in of another Office instance (COMAddIn.Object), handles events of another Office app, or finds a Visio document open in any instance (ROT).
---

# Calls between Office processes

## Expose an object to other processes

Another process gets it by `app.COMAddIns.Item("<registry key name>").Object` (= what `RequestComAddInAutomationService` returns; without the override: `null`).

```csharp
[ComVisible(true), InterfaceType(ComInterfaceType.InterfaceIsDual), Guid("<new GUID>")]
public interface IMyAddInApi { string Snapshot(string path); }

[ComVisible(true), ClassInterface(ClassInterfaceType.None)]
public sealed class MyAddInApi : IMyAddInApi { public string Snapshot(string path) => /* ... */ ""; }

public partial class ThisAddIn
{
    protected override object RequestComAddInAutomationService() => new MyAddInApi();
}
```

Caller (another add-in): late bound, `api.GetType().InvokeMember("Snapshot", BindingFlags.InvokeMethod, null, api, new object[] { path })`.

Measured (Visio Pro C2R x64 build 20625, Debug add-in, 2026-11), caller = the same add-in in a first Visio:

- `new Visio.Application()` started a **new** visible Visio process in 2,013 ms; the add-in's object was there at the first check right after (2,074 ms): the add-in loads before the constructor returns.
- Calls: 0 ms each; returning a string of 1 KB 2 ms, 1 MB 6 ms, 10 MB 50 ms. A snapshot as one JSON string is cheap.
- **The call runs on a worker thread of the called process** (managed thread 8; the UI thread is 1): the .NET object is free-threaded, COM does not send the call to the UI thread. Office objects used there are marshaled to the UI thread and wait while it is busy (seen: a call that opened a document and then read 40 properties took 23 s, the open 1.4 s; cause of the rest not checked). Do the work on the UI thread (e.g. post it, return at once and let the caller poll) rather than in the call (_unverified_ which way works best).

## Find a Visio document open in any instance

Visio registers every open document in the Running Object Table under its full path (object = `Document`), also one opened read-only (`visOpenRO`); `GetActiveObject("Visio.Application")` gives only one instance. Find a document by path like an Excel workbook ([excel-write-table](excel-write-table.md)): enumerate the ROT, compare `IMoniker.GetDisplayName` ignoring case, `GetObject`; `Document.Application` is its instance. Not `GetObject(path)`: it opens a closed file. Verified: Visio 16.0 build 20625, 2026-11, two instances started by script; read-only document opened in a new instance by an Excel VSTO add-in. Not checked: a new document never saved.

## Events of another Office app

A Visio add-in subscribed `Excel.Application.SheetSelectionChange` (Excel started by the add-in):

- `Range("A5").Select()` by a script fired the handler in Visio on managed thread 4, apartment **MTA**, not on Visio's UI thread: .NET event sinks are free-threaded too.
- Visio calls from that handler worked (`Window.CenterViewOnShape` in the own Visio and in a second Visio process; handler 49-79 ms). Same thread rules as above.
- Not checked: a user's click or key in Excel (only `Range.Select` by COM).

Related: `Application.ProcessID` is no Windows process ID ([visio-shapesheet-calls](visio-shapesheet-calls.md)).
