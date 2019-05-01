# Completely Automate Managing Windows Software...Forever
Slides and Demos from my PowerShell Summit 2019 presentation on Chocolatey

Chocolatey-tools module - (https://github.com/dfranciscus/Chocolatey-tools)

## Basic Chocolatey commands:
```
choco list googlechrome -lo
choco uninstall googlechrome -y
choco install googlechrome -y
```

## Internalizing out of date Chocolatey packages and pushing to internal repo:
```
PS C:\> $Outdatedpkgs = Get-ChocoOutdatedPackages
PS C:\> $Outdatedpkgs

Name                 CurrentVersion Version       Pinned
----                 -------------- -------       ------
chocolatey.extension 2.0.1          2.0.2         false
curl                 7.64.0         7.64.1        false
GoogleChrome         73.0.3683.103  74.0.3729.131 false


PS C:\> $intpkgs = Invoke-ChocoInternalizePackage -PackageNames $Outdatedpkgs -Path $Path `
-PurgeWorkingDirectory | Where-Object { $_.Result -Like 'Internalize Success' }
PS C:\Chocotemp> $intpkgs

Name         Result              Version       NuGetpkgs
----         ---the ---              -------       ---------
curl         Internalize Success 7.64.1        C:\Chocotemp\curl.7.64.1.nupkg
GoogleChrome Internalize Success 74.0.3729.131 {C:\Chocotemp\chocolatey-core.extension.1.3.3.nupkg, C:\Chocotemp\Googl...


PS C:\Chocotemp> $Upgradepkgs = Invoke-ChocoUpgradeIntPackage -PackageNames $intpkgs -Path $Path |
 Where-Object {$_.Result -eq 'Upgrade Success'}
PS C:\Chocotemp> Where-Object {$_.Result -eq 'Upgrade Success'}
$Upgradepkgs

Name         Result          Version       NuGetpkgs
----         ------          -------       ---------
curl         Upgrade Success 7.64.1        C:\Chocotemp\curl.7.64.1.nupkg
GoogleChrome Upgrade Success 74.0.3729.131 {C:\Chocotemp\chocolatey-core.extension.1.3.3.nupkg, C:\Chocotemp\GoogleChr...


PS C:\Chocotemp> $Pushed = Push-ChocoIntPackage -PackageNames $Upgradepkgs -Path $Path `
-ApiKey $Api -RepositoryURL $LocalRepo |
Where-Object {$_.Result -like 'Push Success'}
PS C:\Chocotemp> $Pushed

Name                      Result       Version       NuGetPackage
----                      ------       -------       ------------
curl                      Push Success 7.64.1        C:\Chocotemp\curl.7.64.1.nupkg
chocolatey-core.extension Push Success 1.3.3         C:\Chocotemp\chocolatey-core.extension.1.3.3.nupkgpkg
GoogleChrome              Push Success 74.0.3729.131 C:\Chocotemp\GoogleChrome.74.0.3729.131.nupkg
}
```
## Internalizing with connecting commands through the pipeline and triggering an upgrade job:
```
PS C:\Chocotemp> Get-ChocoOutdatedPackages |
Invoke-ChocoInternalizePackage -Path $Path -PurgeWorkingDirectory | Where-Object { $_.Result -Like 'Internalize Success' } |
Invoke-ChocoUpgradeIntPackage -Path $Path | Where-Object {$_.Result -eq 'Upgrade Success'} |
Push-ChocoIntPackage -Path $Path -ApiKey $Api -RepositoryURL $LocalRepo |
Test-ChocoUpgradeTrigger -TriggerPackages 'googlechrome' -UpgradeScriptPath c:\test.ps1 -TriggeredTime '12 PM' -Credential $DomainCred
Creating scheduled task for GoogleChrome

TaskPath                                       TaskName                          State
--------                                       --------                          -----
\                                              Triggered Choco Upgrade           Ready
PS C:\Chocotemp>
```
## Upgrade remote Chocolatey machines with Invoke-ChocoRemoteUpgrade
```
PS C:\Chocotemp> Invoke-ChocoRemoteUpgrade -ComputerName winclient,winclient2 -Credential $DomainCred `
 -AdditionalPackages firefox -RebootifPending | Select-Object -Property Name,Result,Computer |
 Format-List


Name     : googlechrome
Result   : Failed
Computer : WINCLIENT2

Name     : googlechrome
Result   : Failed
Computer : WINCLIENT

Name     : firefox
Result   : Success
Computer : WINCLIENT2

Name     : firefox
Result   : Success
Computer : WINCLIENT
```
## Upgrade remote Chocolatey clients with Invoke-BoxstarterRemoteUpgrade:
```
Invoke-BoxStarterRemoteUpgrade -ComputerName winclient2 -Credential $DomainCred `
-AdditionalPackages curl,git -ExcludedPackages jre8 `
-ScriptPath C:\Windows\Temp\BoxstarterUpgrade.txt

Invoke-Command -ComputerName winclient2 -Credential $DomainCred -ScriptBlock {
    Get-Content -Path C:\Windows\Temp\BoxstarterUpgrade.txt
}
choco upgrade curl -r -y --timeout=600
choco upgrade git -r -y --timeout=600
choco upgrade googlechrome -r -y --timeout=600
```
