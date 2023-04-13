#ErrorMessage #PowerShell #Troubleshoot #Module

<br>

## Error Message

WARNING: Unable to resolve package source ‘https://www.powershellgallery.com/api/v2’.  
PackageManagement\Install-Package : No match was found for the specified search criteria and  
module name ‘PnP.PowerShell’. Try Get-PSRepository to see all available registered module  
repositories.  
At C:\Program Files\WindowsPowerShell\Modules\PowerShellGet\1.0.0.1\PSModule.psm1:1772 char:21

-   … $null = PackageManagement\Install-Package @PSBoundParameters
    -   CategoryInfo : ObjectNotFound: (Microsoft.Power….InstallPackage:InstallPacka  
        ge) [Install-Package], Exception
    -   FullyQualifiedErrorId : NoMatchFoundForCriteria,Microsoft.PowerShell.PackageManagement.  
        Cmdlets.InstallPackage

<br>

## Set TLS 1.2 on your PowerShell

```powershell
[Net.ServicePointManager]::SecurityProtocol = [Net.SecurityProtocolType]::Tls12
```