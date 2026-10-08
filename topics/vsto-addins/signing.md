---
title: Signing
description: Read when signing a VSTO add-in (Authenticode, manifests, MSI), choosing the MSBuild hook for signtool, separating debug/release certificates, or when an expired test certificate blocks loading.
---

# Signing

Three independent signatures:

| Artifact | Kind | Controlled by |
|---|---|---|
| `.vsto` + `.dll.manifest` | XML signature (ClickOnce) | `.csproj`: `SignManifests`, `ManifestCertificateThumbprint`, `ManifestKeyFile` |
| Add-in assembly | Authenticode | own `signtool` call |
| MSI | Authenticode | own `signtool` call in the `.wixproj` |

## Sign the assembly before the manifests are generated

`.dll.manifest` stores size and hash of each file. Authenticode changes both (measured 78,848 → 83,024 bytes). If you sign after manifest generation, the add-in silently does not load.

Hook: `VisualStudioForApplicationsBuild` (from `Microsoft.VisualStudio.Tools.Office.targets`) generates and signs the manifests. `GenerateApplicationManifest` does not exist in VSTO projects, and MSBuild silently ignores `BeforeTargets` on an unknown target.

```xml
<Target Name="SignAddinAssembly"
        Condition="'$(SignReleaseArtifacts)' == 'true'"
        AfterTargets="CopyFilesToOutputDirectory"
        BeforeTargets="VisualStudioForApplicationsBuild">
  <Exec Command="&quot;$(SignToolPath)&quot; sign /sha1 $(Thumbprint) /fd SHA256 /tr $(TimestampUrl) /td SHA256 &quot;$(TargetPath)&quot;" />
</Target>
```

Measured order: `CopyFilesToOutputDirectory` → (sign here) → `VisualStudioForApplicationsBuild` → `RegisterOfficeAddin`. The MSI is signed after its build and is not order-sensitive.

Check a hook without editing the project. Pass a probe targets file and look where its message appears:

```xml
<!-- probe.targets -->
<Project>
  <Target Name="Probe" AfterTargets="CopyFilesToOutputDirectory" BeforeTargets="VisualStudioForApplicationsBuild">
    <Message Importance="high" Text="PROBE" />
  </Target>
</Project>
```

```powershell
msbuild MyAddin.csproj /t:Rebuild /v:normal /p:CustomAfterMicrosoftCommonTargets=C:\path\probe.targets
```

## Debug vs release

- Token/cloud certificates (e.g. Certum SimplySign) need a confirmation per signature, which is unusable for dev builds. Debug: VS self-signed test certificate for manifests, no Authenticode. Release: real certificate.
- Safe when installed to `Program Files`: there the VSTO runtime grants FullTrust by location, without the inclusion-list prompt. Not when the location check is bypassed, see expired certificate below.
- Put thumbprint and a switch (`SignReleaseArtifacts`) in a shared `Signing.props` imported by `.csproj` and `.wixproj`. Override with `/p:SignReleaseArtifacts=false` to test release builds without the token.
- Release manifests with a CSP-bound key: set no `ManifestKeyFile`. MSBuild's `SignFile` worked with the SimplySign CSP key.

## Expired test certificate

VS test certificate `<AssemblyName>_TemporaryKey.pfx` is valid for a limited time (about 1 year) and is not renewed on build. Builds keep signing with it without warning. When it has expired, Office reports "Publisher cannot be verified" even for `Program Files` installs, so the location-based FullTrust did not help.

```powershell
(New-Object System.Security.Cryptography.X509Certificates.X509Certificate2("<AssemblyName>_TemporaryKey.pfx")).NotAfter
```

Fix: VS project properties → Signing → "Create Test Certificate", or:

```powershell
$cert = New-SelfSignedCertificate -Type CodeSigningCert -Subject "CN=<Name>" -KeyExportPolicy Exportable `
  -KeySpec Signature -KeyUsage DigitalSignature -CertStoreLocation Cert:\CurrentUser\My -NotAfter (Get-Date).AddYears(5)
Export-PfxCertificate -Cert $cert -FilePath "<AssemblyName>_TemporaryKey.pfx" -Password (New-Object System.Security.SecureString) -Force
```

Then set `<ManifestCertificateThumbprint>` to `$cert.Thumbprint` and rebuild.

_Unverified:_ whether an unsigned manifest behaves differently from an expired one.

## Verify

```powershell
# manifest size matches the signed DLL (smaller = signed after manifest generation)
$n = Select-Xml -Path "$bin\MyAddin.dll.manifest" -XPath "//*[local-name()='dependentAssembly'][@dependencyType='install']" |
     Where-Object { $_.Node.codebase -eq 'MyAddin.dll' }
[int]$n.Node.size -eq (Get-Item "$bin\MyAddin.dll").Length

$s = Get-AuthenticodeSignature $path
$s.Status; $s.TimeStamperCertificate   # Valid; not empty (no timestamp = invalid once the cert expires)
```

Also decode the manifest's `<X509Certificate>` to confirm release builds do not still use the test certificate.
