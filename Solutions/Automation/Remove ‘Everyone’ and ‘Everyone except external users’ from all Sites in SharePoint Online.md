#Automation #PowerShell #PnP #SiteCollection #Subsite #Everyone #EveryoneExceptExternalUsers

<br>

```powershell
#Define Parameters
$AdminSiteURL= "https://DOMAIN-admin.sharepoint.com"
$ItemCounter = 0

#Get Credentials to connect
$Cred  = Get-Credential

#Connect to Services
Connect-PnPOnline -Url $AdminSiteURL –Credential $Cred

#Get all Sites and itinerate
$Sites = Get-PnPTenantSite | Where{ ($_.Title -notlike "" -and $_.Template -notlike "*Redirect*" -and $_.Url -notlike "*my.sharepoint.com*") }
Write-Host -f Cyan $Sites.Count
ForEach($Site in $Sites){

    #Status notification
    $ItemCounter++
    $ItemProcess = [math]::Round($ItemCounter/$Sites.Count*100,1)
    Write-Progress -PercentComplete $ItemProcess -Activity "Processing $($ItemProcess)%" -Status "Site '$($Site.Url)"

    Try{
        # REMOVE GROUP FROM SITE COLLECTION
        Connect-PnPOnline -Url $Site.url –Credential $Cred

        Remove-PnPUser -Identity "Everyone" -Force
        Remove-PnPUser -Identity "Everyone except external users" -Force

        Try{
            # REMOVE GROUP FROM SUBSITES
            $SubSites = Get-PnPSubWeb -Recurse -Includes HasUniqueRoleAssignments
            ForEach($SubSite in $SubSites){

                # Subsite with unique permissions
                if ($SubSite.HasUniqueRoleAssignments){
                    Connect-PnPOnline -Url $SubSite.url –Credential $Cred

                    Remove-PnPUser -Identity "Everyone" -Force
                    Remove-PnPUser -Identity "Everyone except external users" -Force
                }
                Write-Host -f Green $SubSite.url"COMPLETED"
            }
        }
        Catch{
            Write-Host -f Red $SubSite.url"ERROR"
        }
        
        Write-Host -f Green $Site.url"COMPLETED"
    }
    Catch{
        Write-Host -f Red $Site.url"ERROR"
    }
}
#Close status notification
Write-Progress -Activity "Processing $($ItemProcess)%" -Status "Site '$($Site.URL)"
```