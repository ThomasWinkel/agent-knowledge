---
title: Office icons for ribbon buttons
description: Read when choosing a built-in Office icon (imageMso, OfficeImageId) for a custom ribbon button - check that an ID exists, view the icons, no icon for some commands.
---

# Office icons for ribbon buttons

A custom ribbon button can show a built-in Office icon: `imageMso="<id>"` (ribbon XML), `OfficeImageId = "<id>"` (VSTO ribbon designer). An unknown ID shows no icon and no error.

## Check that an ID exists

`Application.CommandBars.GetImageMso(id, 32, 32)` from another process (PowerShell, COM automation) returns no picture: the `IPictureDisp` does not cross the process. The error still tells known from unknown IDs:

```powershell
$xl = New-Object -ComObject Excel.Application
foreach ($id in 'SelectAll', 'NoSuchId') {
    try { $null = $xl.CommandBars.GetImageMso($id, 32, 32); "$id ok" }
    catch { "$id -> $($_.Exception.HResult.ToString('X'))" }   # 8000FFFF = exists, 80070057 = unknown
}
$xl.Quit()
```

- Known ID: `0x8000FFFF` `E_UNEXPECTED` ("Catastrophic failure"). Unknown ID: "Value does not fall within the expected range" (`E_INVALIDARG`).
- The ID set is shared by the Office apps: an ID checked in Excel also works in Visio (seen: Visio icon IDs `ObjectsAlignMiddleVertical`, `SelectionPane`).

## View the icons

- In-process only (VBA `stdole.SavePicture` of `GetImageMso`, or code inside an add-in). VBA by automation needs "Trust access to the VBA project object model", a security setting: ask the user before changing it.
- Without that: the public list https://bert-toolkit.com/imagemso-list.html shows name and icon. The icons are one sprite `https://bert-toolkit.com/img/mso-composite-16.png` (16 px wide, one icon every 16 px); the list gives each ID's offset (`background-position:0 -<y>px`). Crop the rows of your candidates to compare them. Which Office version the sprite shows is not stated: check the icon in the app.

## Gotchas

- Not every command has an icon. Example: no ID contains "Invert" or "Reverse" for a selection; pick a related one (`SelectionPane`: several shapes, one marked).

Verified with Office 2019 C2R x64, Windows PowerShell 5.1 (2026-10). _Unverified:_ the exact unknown-ID HRESULT on other versions; whether the sprite matches Office 2019 icons.
