# Collect Windows OS build details in Azure App Service

This article explains how to collect Windows operating system information exposed to a Windows-based Azure App Service environment using a read-only PowerShell script. The output helps correlate a diagnostic session with an app instance, a UTC timestamp, and the Windows build and update revision visible to that session.

> **Scope:** This is a point-in-time diagnostic snapshot of the environment where the script runs. It is not a host inventory, an installed-update report, or proof that a particular security fix is present or absent. For Windows containers, information collected inside the container describes that environment and must not be treated as the underlying host's OS version.

## Before you begin

- Use a Windows-based App Service app and a PowerShell console in the environment you want to inspect. This script does not apply to Linux apps.
- You need authorized access to the app's Kudu (Advanced Tools) site. Microsoft documents the required permissions in the [Kudu service overview](https://learn.microsoft.com/en-us/azure/app-service/resources-kudu).
- The registry path must be readable in that environment. If access is blocked or the key is unavailable, the script intentionally stops rather than returning an apparently successful report.

The script does not change the registry, app settings, or files; restart the app; or send results over the network. Its output includes app and instance identifiers, computer names, and paths. Review and redact those values before sharing outside an approved support channel.

## Start with the Kudu Environment page

> **Note:** Generally, you can find most of the information needed for routine environment troubleshooting on the Kudu Environment page: `https://<SiteName>.scm.azurewebsites.net/Env.cshtml`.

Replace `<SiteName>` with your app's name and sign in with an authorized account. You can also open **Advanced Tools > Go** from the app or slot in the Azure portal, then select **Environment** in Kudu. For deployment slots, App Service Environments, or other hostname variations, use the actual SCM hostname opened by the portal and append `/Env.cshtml`.

The page provides system information such as the process-reported OS version, machine name, instance ID, system directory, and OS/process architecture, together with app settings, connection strings, environment variables, and other diagnostic context.

Use this page first when you only need a quick environment check. Use the script below when you need the registry-reported Windows build and update revision (`UBR`), or a consistent, timestamped report. The Environment page does not necessarily expose every script field or establish the underlying host's patch status; its OS version is process-reported.

**Protect sensitive information:** This page can expose secrets in app settings, connection strings, environment variables, and request details. Do not share a full page export or unredacted screenshot, and do not assume every secret is masked. Share only the reviewed fields required for the support case.

## Run the script

1. **Open Kudu Console** at `https://<SiteName>.scm.azurewebsites.net` and sign in with an authorized account. Use the actual SCM hostname for the target app or deployment slot.
2. In Kudu, select **Debug console > PowerShell**, where available.
3. Paste and run the complete script below. Copy the output and record the app, slot, and Azure region in your support notes.

Run the script in the target environment, not on your workstation or in Azure Cloud Shell. Otherwise, it reports information from that other environment. For custom Windows containers, do not assume that the Kudu console is inside your application container; use an authorized PowerShell session inside the target container when container information is required.

### PowerShell script

```powershell
$registryPath = 'HKLM:\SOFTWARE\Microsoft\Windows NT\CurrentVersion'

# Stop explicitly if this environment does not permit registry access.
$os = Get-ItemProperty -Path $registryPath -ErrorAction Stop

$build = $os.CurrentBuildNumber
$revision = $os.UBR

$buildAndRevision = if (
    -not [string]::IsNullOrWhiteSpace($build) -and
    $null -ne $revision
) {
    '{0}.{1}' -f $build, $revision
} else {
    'Unavailable: build number or update revision is not exposed'
}

$details = [ordered]@{
    CollectedAtUTC      = [DateTime]::UtcNow.ToString('o')
    AppName             = $env:WEBSITE_SITE_NAME
    InstanceId          = $env:WEBSITE_INSTANCE_ID
    ComputerName        = $env:COMPUTERNAME

    OSProductName       = $os.ProductName
    ProcessOSVersion    = [Environment]::OSVersion.VersionString
    DisplayVersion     = $os.DisplayVersion
    InstallationType   = $os.InstallationType
    BaseBuild          = $build
    UpdateBuildRevision = $revision
    BuildAndRevision   = $buildAndRevision
    BuildLab           = $os.BuildLab
    BuildLabEx         = $os.BuildLabEx
    Is64BitOS          = [Environment]::Is64BitOperatingSystem
    Is64BitProcess     = [Environment]::Is64BitProcess

    RegistrySource     = $registryPath
    WindowsDirectory   = $os.WindowsDirectory
    SystemRoot         = $env:SystemRoot
    SystemDirectory    = [Environment]::SystemDirectory
    AppServiceHome     = $env:HOME
}

# Make unavailable fields explicit instead of displaying empty values.
foreach ($key in @($details.Keys)) {
    if ($null -eq $details[$key] -or
        [string]::IsNullOrWhiteSpace([string]$details[$key])) {
        $details[$key] = 'Not exposed in this environment'
    }
}

[PSCustomObject]$details | Format-List
```

The code uses a single backslash in the registry path and literal underscores in the environment-variable names. Copy it directly from the code block without adding Markdown escape characters.

### Sample output

The following example is adapted from a sample run, with the app name, instance ID, and computer name replaced by placeholders. Your timestamp, build/revision, paths, and other values will vary. This is illustrative output, not a required OS version or a statement of the platform's current patch level.

```text
CollectedAtUTC      : 2026-11-02T06:29:00.1846514Z
AppName             : <app-name>
InstanceId          : <instance-id>
ComputerName        : <computer-name>
OSProductName       : Windows Server 2022 Datacenter Azure Edition
ProcessOSVersion    : Microsoft Windows NT 10.0.20348.0
DisplayVersion      : 21H2
InstallationType    : Server
BaseBuild           : 20348
UpdateBuildRevision : 5773
BuildAndRevision    : 20348.5773
BuildLab            : 20348.fe_release.210507-1500
BuildLabEx          : 20348.1.amd64fre.fe_release.210507-1500
Is64BitOS           : True
Is64BitProcess      : False
RegistrySource      : HKLM:\SOFTWARE\Microsoft\Windows NT\CurrentVersion
WindowsDirectory    : Not exposed in this environment
SystemRoot          : C:\Windows
SystemDirectory     : C:\Windows\system32
AppServiceHome      : C:\home
```

In this example, the registry build and revision combine to `20348.5773`, while the process-reported OS version shows `10.0.20348.0`. Retain both values rather than treating them as interchangeable. `Is64BitProcess : False` alongside `Is64BitOS : True` indicates a 32-bit PowerShell process on a 64-bit OS; it does not, by itself, establish the application worker's bitness.

## Understand the output

| Field | How to use it |
| --- | --- |
| `CollectedAtUTC` | UTC collection timestamp in round-trip ISO 8601 format. Use it to correlate with logs and incident times. |
| `AppName`, `InstanceId`, `ComputerName` | Context exposed to the current session. Use these values to identify the environment being inspected; missing values are explicitly labeled. |
| `BaseBuild` | Windows build number read from `CurrentBuildNumber` in the registry. |
| `UpdateBuildRevision` | Update build revision read from `UBR`. A value of `0` is valid and is not treated as missing. |
| `BuildAndRevision` | Combines the two registry values as `<build>.<revision>`. If either is missing, the script says the combined value is unavailable. |
| `OSProductName`, `DisplayVersion`, `InstallationType` | Descriptive registry metadata. Availability varies; do not identify the OS solely from a product name or display-version string. |
| `ProcessOSVersion` | The OS version reported to the current PowerShell/.NET process. Process-reported and registry-reported information can differ; retain both for troubleshooting. |
| `BuildLab`, `BuildLabEx` | Additional build metadata, when exposed. These values are not an installed-update history. |
| `Is64BitOS`, `Is64BitProcess` | OS and process architecture indicators. A 32-bit process can run on a 64-bit OS. |
| `RegistrySource`, directory fields | Identifies the registry source and paths visible to the session. Paths alone do not establish host identity or patch status. |

`Not exposed in this environment` means that a field was missing, null, empty, or whitespace. It does not by itself indicate an application failure. For example, `DisplayVersion` may not be populated even when build information is available.

## Important interpretation limits

- **One run covers one execution context.** Do not assume a Kudu result represents every scaled-out application instance. Record the instance information and repeat collection in the relevant instance contexts when investigating instance-specific behavior.
- **Values can change over time.** Repeat collection when needed and retain the timestamp and instance identifiers with each result.
- **The script does not enumerate installed KBs.** A build/revision pair can support further investigation, but confirming a specific fix requires the correct Windows product and servicing context, applicable Microsoft release documentation, and, where needed, Azure support.
- **App Service is a managed platform.** Use this script to observe exposed information, not to change platform registry values or install host OS updates. See [Operating system functionality in Azure App Service](https://learn.microsoft.com/en-us/azure/app-service/operating-system-functionality) for access boundaries.

## Troubleshooting

| Symptom | Recommended action |
| --- | --- |
| Kudu cannot be opened | Verify that you selected the correct app or slot, have the documented Kudu permissions, and can reach the SCM endpoint under the app's networking rules. Do not weaken access controls just to collect this output. |
| `HKLM:` is unavailable or the registry read fails | Confirm that you are using PowerShell in a Windows environment. Preserve the error text. If registry access is restricted, use an approved diagnostic route or contact Azure support; do not bypass the restriction. |
| App or instance identifiers are not exposed | Check where the script was executed. A local shell, Cloud Shell, or a different diagnostic context may not expose the target app's variables. |
| `BuildAndRevision` is unavailable | One or both required registry values were not exposed. Keep the remaining output, but do not infer a revision or substitute a process-version string as an equivalent. |
| Results differ across sessions | Compare collection times, app/slot, instance identifiers, process architecture, and whether collection occurred in Kudu or inside a container before drawing conclusions. |

For a support case, provide the reviewed script output, the app and slot, Azure region, incident time in UTC, collection context, and any exact error text through your approved support channel. Do not include credentials, publishing profiles, connection strings, or unrelated environment-variable dumps.

## Kudu API knowledge base

For the full documented Kudu REST API catalog, including HTTP methods, request formats, and examples, use the [Kudu REST API reference](https://github.com/projectkudu/kudu/wiki/REST-API). The [Kudu wiki](https://github.com/projectkudu/kudu/wiki) provides the broader knowledge base for console access, deployments, diagnostics, and troubleshooting.

API URLs use the same SCM hostname as the Kudu portal:

```text
Kudu portal:       https://<SiteName>.scm.azurewebsites.net/
Environment page: https://<SiteName>.scm.azurewebsites.net/Env.cshtml
Example API:       https://<SiteName>.scm.azurewebsites.net/api/environment
```

Use the target app or slot's actual SCM hostname. `/Env.cshtml` is a browser page, not a REST API. The reference documentation is the starting point for discovering the API surface; do not assume that opening `/api` will list every available API.

### Read-only API quick reference

These are selected documented endpoints, not an exhaustive inventory:

| HTTP method and path | Purpose | Reference |
| --- | --- | --- |
| `GET /api/environment` | Returns Kudu version information; it is not equivalent to the full Environment page or the registry report above. | [Environment API](https://github.com/projectkudu/kudu/wiki/REST-API#environment) |
| `GET /api/deployments` | Lists deployment history recorded by Kudu. | [Deployment API](https://github.com/projectkudu/kudu/wiki/REST-API#deployment) |
| `GET /api/deployments/{id}/log` | Lists log entries for a specific deployment. | [Deployment API](https://github.com/projectkudu/kudu/wiki/REST-API#deployment) |
| `GET /api/vfs/{path}/` | Lists a directory; keep the trailing slash. Without it, `GET /api/vfs/{path}` reads a file. | [Virtual file system API](https://github.com/projectkudu/kudu/wiki/REST-API#vfs) |
| `GET /api/dump` | Downloads diagnostic logs as a ZIP file. Review its contents for sensitive data before sharing. | [Diagnostics API](https://github.com/projectkudu/kudu/wiki/REST-API#diagnostics) |
| `GET /api/siteextensions` | Lists installed site extensions, where supported. | [Site extensions API](https://github.com/projectkudu/kudu/wiki/REST-API#siteextensions) |

### Access and safety

Open the Kudu portal through the Azure portal for interactive, authorized browser access. For scripted API access, follow the [Kudu access and authentication guidance](https://github.com/projectkudu/kudu/wiki/Accessing-the-kudu-service) and your organization's approved authentication method. Browser sign-in does not automatically authenticate a separate PowerShell or curl session. Some older examples use publishing credentials with Basic authentication; those examples will not work when SCM Basic authentication is disabled. Do not enable it or weaken network restrictions solely to follow an example.

API availability and behavior vary by platform, hosting configuration, and deployed Kudu version. Some wiki examples describe older versions. A missing endpoint is not a reason to bypass access controls.

The full API reference also documents operations that execute commands, deploy code, change settings, or delete files and repositories. Those operations are not read-only diagnostics and are not required for this article. Even read-only responses can contain application data or secrets; do not paste credentials into shared commands or publish unredacted API responses.

## References

- [Kudu service overview and access permissions](https://learn.microsoft.com/en-us/azure/app-service/resources-kudu)
- [Operating system functionality and access boundaries in Azure App Service](https://learn.microsoft.com/en-us/azure/app-service/operating-system-functionality)
- [Kudu REST API reference](https://github.com/projectkudu/kudu/wiki/REST-API)
- [Kudu access and authentication guidance](https://github.com/projectkudu/kudu/wiki/Accessing-the-kudu-service)
- [Kudu knowledge base](https://github.com/projectkudu/kudu/wiki)

This article is a diagnostic aid, not an official statement of a particular App Service instance's OS servicing or compliance status.
