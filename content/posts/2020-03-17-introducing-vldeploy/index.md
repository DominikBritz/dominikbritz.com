---
author: Dominik Britz
date: "2020-03-17"
title: Introducing vlDeploy - Deploy Applications With PowerShell Remotely
postSlug: introducing-vldeploy
featured: false
draft: false
tags:
  - powershell
  - euc
# description:
#   Deploy Applications With PowerShell Remotely
comments: true
cover:
  image: "https://dominikbritz.com/posts/2020-03-17-introducing-vldeploy/images/featured2.png"
  alt: gears
  responsiveImages: false
  relative: false
---

Managing applications on Windows in enterprises is complex and cumbersome. Admins are using a variety of tools to install, uninstall or reconfigure applications silently. The most popular tool is propably Microsoft's ConfigMgr.

While ConfigMgr (and others) makes sense for mid to large enterprises, the management is too time-consuming for smaller firms. I work in such a smaller firm. We have a lab to test our own applications [uberAgent](https://uberagent.com) on several operating systems. These lab machines don't only run uberAgent, of course, they also run standard applications like text editors, browsers, and more complex things.

While we are using Microsoft MDT to install the OS and set everything up, all these applications need some sort of installing/updating mechanism. We use chocolatey for most of them. However, not all packages can be deployed with chocolatey. Some need special treatment, pre-install or post-install actions, or there is simply no chocolatey package available.

For these 'special treatment' applications we developed our own PowerShell module called `vlDeploy`, which I'm happy to share publically.

## Introducing vlDeploy
`vlDeploy` is a PowerShell module publically available in the PowerShell gallery. It allows the installation of applications locally, or even better, remotely. The remoting capability enables administrators to push applications to multiple machines.

Here is a list of its features:
- Gets a list of installed applications locally, or, from a remote machine
- Installs applications locally, or, on remote machines
- Uninstalls applications locally, or, on remote machines
- Support for `.exe`, `.msi`, and `.ps1` installers
  - Very cool: throw the `.msi` installer at the module and it will take care of installing the application silently. Same is true for `.ps1` files.
- Support for URLs as installer
	- The installer will be downloaded and then deployed
- Powershell pipeline support
- Valid exit codes handover
- Reboot after success

Is it a full blown application distribution and everything else can get tossed to the void? No, of course not. It has its use-cases which I'm describing in the following.

Combining `vlDeploy` with other tools gives you a very powerful PowerShell-based installation framework. See chapter *Combine vlDeploy With Other Tools* for details.

## Prerequisites

Before you begin, ensure you have met the following requirements:

* At least PowerShell version 5.0
* Windows operating system

## Installing vlDeploy

The module is availabe in the PowerShell Gallery. To install `vlDeploy`, follow these steps:

```powershell
Install-Module vlDeploy
```

## Using vlDeploy

`vlDeploy` exposes three functions to list installed applications, install applications, and uninstall applications. 

To see what's possible with each, use:

```powershell
Get-Help Install-vlApplication -full
Get-Help Uninstall-vlApplication -full
Get-Help Get-vlInstalledApplication -full
```

### Examples

Simple application installation on a remote machine.

```powershell
$cred = Get-Credential
Install-vlApplication -Computer PC1 -Sourcefiles 'C:\apps\source' -Installer Setup.exe -InstallerArguments '/silent /noreboot' -Credential $cred
```

Application installation on a remote machine with a PowerShell script downloaded from the Internet.

```powershell
$cred = Get-Credential
Install-vlApplication -Computer PC1 -Installer 'https://somewebsite.com/apps/Install.ps1' -Credential $cred
```


`Install-vlApplication` accepts pipeline input for the `-Computer` parameter , which makes it easy to mass-deploy applications. It also recognizes `.msi` installer files and builds the full install command automatically.

```powershell
$cred = Get-Credential
'PC1', 'PC2', 'PC3' | Install-vlApplication -Sourcefiles 'C:\apps\source' -Installer Setup.msi -Credential $cred
```


Get the uninstall string of an application from a remote machine and uninstall it there. Valid exit codes are `0`, `3010`, and `1` (default `0` and `3010`). 

```powershell
$cred = Get-Credential
Get-vlInstalledApplication -Computer PC1 -Name 'Google Chrome' -Credential $cred | Uninstall-vlApplication -Credential $cred -ValidExitCodes 0,3010,1
```

Get the uninstall string of an application from a remote machine and uninstall it on multiple machines. Reboot every machine if uninstallation was successful.

```powershell
$cred = Get-Credential
$UninstallString = (Get-vlInstalledApplication -Computer PC1 -Name 'Google Chrome' -Credential $cred).UninstallString
'PC1', 'PC2', 'PC3' | Uninstall-vlApplication -UninstallString $UninstallString -Credential $cred -RebootAfterSuccess
```

## Combine vlDeploy With Other Tools
You can combine `vlDeploy` very nicely with other PowerShell tools. The combination with the following tools gives you a very powerful PowerShell-based installation framework

### Evergreen
[Evergreen](https://github.com/aaronparker/Evergreen) is a PowerShell module actively developed by [Aaron Parker](https://stealthpuppy.com/) to return the latest version and download URLs for a set of common enterprise applications.

As of 2020-03-02, the following applications are included:

```
AdobeAcrobatReaderDC
Atom
BISF
CitrixAppLayeringFeed
CitrixApplicationDeliveryManagementFeed
CitrixEndpointManagementFeed
CitrixGatewayFeed
CitrixHypervisorFeed
CitrixLicensingFeed
CitrixReceiverFeed
CitrixSdwanFeed
CitrixVirtualAppsDesktopsFeed
CitrixWorkspaceApp
CitrixWorkspaceAppFeed
CitrixXenServerTools
ControlUpAgent
Cyberduck
FileZilla
FoxitReader
GitForWindows
Good
GoogleChrome
Greenshot
JamTreeSizeFree
JamTreeSizeProfessional
Java8
LibreOffice
MicrosoftEdge
MicrosoftFSLogixApps
MicrosoftOffice
MicrosoftPowerShellCore
MicrosoftSQLServerManagementStudio
MicrosoftSsms
MicrosoftTeams
MicrosoftVisualStudioCode
MozillaFirefox
mRemoteNG
NotepadPlusPlus
OpenJDK
OracleJava8
OracleVirtualBox
PaintDotNet
ScooterBeyondCompare
ShareX
TeamViewer
VideoLanVlcPlayer
VMwareTools
WinMerge
Zoom
```

After installing the module via `Install-Module Evergreen` you can see which applications are available with `(Get-Command -Module Evergreen).Name -replace 'Get-','' | Sort-Object`.

In combination with `vlDeploy` you can deploy applications to multiple hosts without having to maintain a fileshare with dozens of sources.

Below is a simple example using `vlDeploy` in combination with `Evergreen` to install Microsoft Edge.

```powershell
# Variables
$Computers = "PC1", "PC2"
$cred = Get-Credential

# Install modules
$Modules = @("Evergreen", "vlDeploy")
Foreach ($Module in $Modules) {
    If (-not(Get-Module $Module)) {
        Install-Module $Module -Force
        Import-Module $Module
    }
    Else {
        Update-Module $Module
        Import-Module $Module
    }

}

# Get setup URI
$App = Get-MicrosoftEdge | Where-Object Platform -eq 'Windows' | Where-Object Product -eq 'Stable' | Where-Object Architecture -eq 'x64' | Select-Object -First 1

# Install
Install-vlApplication -Installer "$($App.URI)" -Computer $Computers -Credential $cred
```

I recommend having a look at `Evergreen` in more detail. It also includes silent installer arguments as well as pre-install and post-install actions. [Here](https://github.com/aaronparker/Evergreen/blob/master/scripts/Install-AdobeReaderDC.ps1) is nice example for Adobe Reader.

### XenAppBlog Automation Framework Applications
The [XenAppBlog Automation Framework Applications](https://github.com/haavarstein/Applications) by [Trond Eirik Haavarstein](https://xenappblog.com/blog/) are scripts that install applications without the need to maintain installer sources. The framework is using `Evergreen` in some scripts as well. 

Install VMware Tools on multiple computers and reboot without having to think about managing installers.

```powershell
# Variables
$Computers = "PC1","PC2"
$cred = Get-Credential

# Install app
Install-vlApplication -Installer 'https://raw.githubusercontent.com/haavarstein/Applications/master/VMware/Tools/Install.ps1' -Computer $Computers -Credential $cred -RebootAfterSuccess
```

### PowerShell App Deployment Toolkit
`Evergreen` and the *XenAppBlog Automation Framework Applications* are limited to a set of common enterprise applications. You won't find your very special LOB application in there. LOB applications are best deployed with the [PowerShell App Deployment Toolkit](https://github.com/PSAppDeployToolkit/PSAppDeployToolkit).

> The PowerShell App Deployment Toolkit provides a set of functions to perform common application deployment tasks and to interact with the user during a deployment. It simplifies the complex scripting challenges of deploying applications in the enterprise, provides a consistent deployment experience and improves installation success rates. *Source: https://github.com/PSAppDeployToolkit/PSAppDeployToolkit*

The installation logic is defined in the file `Deploy-Application.ps1`. That makes the integration with `vlDeploy` very easy.

```powershell
# Variables
$Computers = "PC1","PC2"
$cred = Get-Credential
$Sourcefiles = "\\server\share\apps"

# Install apps
$Folders = (Get-ChildItem $Sourcefiles -Directory).FullName
Foreach ($Folder in $Folders) {
    Install-vlApplication -Sourcefiles $Folder -Installer 'Deploy-Application.ps1' -Computer $Computers -Credential $cred
}
```

A step-by-step guide for the PowerShell App Deployment Toolkit is available [here from replicajunction](https://replicajunction.github.io/2015/04/07/installing-applications-with-psadt-part1/).

## Development And Contributing
The module has a feature set tailored to the needs we have in our company. If you need something else, please contribute at [GitHub](https://github.com/vastlimits/vlDeploy).

A list of features I would like to add, but there was not enough time yet:
- Run install/uninstall as PowerShell jobs to benefit from parallelism
    - Perhaps use PowerShell 7 and `Foreach-Object -Parallel`. [More information](https://devblogs.microsoft.com/powershell/powershell-foreach-object-parallel-feature/).
- Add support for `.bat`,`.cmd`, `.msp`, `.mst`, and `.msu` installers
- Output only applications which are listed in *Add/Remove programs* by default. Output everything only when desired.