#Automation #OneDrive #SharePointOnline #PnP #PowerShell 
<br>

## SharePoint Online Management Shell

```powershell
# Define Parameters
$AdminSiteURL = "https://DOMAIN-admin.sharepoint.com"
$SiteCollAdmin = "Admin@Email.com"
$UserToDelete = "UserToDelete@email.com"
$ItemCounter = 0

# Get Credentials
$Cred = Get-Credential

# Connect to SharePoint Online Admin Center
Connect-SPOService -Url $AdminSiteURL –Credential $Cred

# Get all Site collections and iterate
$Sites = Get-SPOSite -Limit ALL -IncludePersonalSite $True | Where{ ($_.Title -notlike "" -and $_.Template -notlike "*Redirect*") }
Write-Host -f Cyan 'Total number of Site Collections:'$Sites.Count
ForEach($Site in $Sites){

    # Status notification
    $ItemCounter++
    $ItemProcess = [math]::Round($ItemCounter/$Sites.Count*100,1)
    Write-Progress -PercentComplete $ItemProcess -Activity "Processing $($ItemProcess)%" -Status "Site '$($Site.URL)"
    
    Try{
        Write-host -f Yellow $Site.URL
        
        # Add Site Collection Admin
        Set-SPOUser -Site $Site.Url -LoginName $SiteCollAdmin -IsSiteCollectionAdmin $True
        
        # Remove user from Site Collection
        Remove-SPOUser -Site $Site.Url -LoginName $UserToDelete
        
        # Remove Site Collection Admin
        Set-SPOUser -Site $Site.Url -LoginName $SiteCollAdmin -IsSiteCollectionAdmin $False
        }

    Catch{
        }

    }
# Close status notification
Write-Progress -Activity "Processing $($ItemProcess)%" -Status "Site '$($Site.URL)"

Write-Host -b Green "Finished removing user as from Site Collection!"
```

<br>

<br>

## PnP PowerShell

```powershell
#################################################################
# DEFINE PARAMETERS FOR THE CASE
#################################################################
$SPOAdminURL = "https://<DOMAIN>-admin.sharepoint.com/"
$AdminUPN = "ADMIN@EMAIL.com"
$UserToDelete = "USER@EMAIL.com"



#################################################################
# REPORT AND LOGS FUNCTIONS
#################################################################

Function Add-ReportRecord($Site, $User) {
    
    $Record = New-Object PSObject -Property ([ordered]@{
        "SiteName" = $Site.Title
        "SiteUrl" = $Site.Url
        "UserName" = $User.Title
        "UserEmail" = $User.Email
        })
    
    $Record | Export-Csv -Path $ReportOutput -NoTypeInformation -Append
}

Function Add-ScriptLog($Color, $Msg) {
    Write-host -f $Color $Msg
    $Date = Get-Date -Format "yyyy/MM/dd HH:mm"
    $Msg = $Date + " - " + $Msg
    Add-Content -Path $LogsOutput -Value $Msg
}

# Create Report location
$FolderPath = "$Env:USERPROFILE\Documents\NovaPoint\Automation\"
$Date = Get-Date -Format "yyyyMMddHHmmss"
$ReportName = "OwnersMembersReport"
$FolderName = $Date + "_" + $ReportName
New-Item -Path $FolderPath -Name $FolderName -ItemType "directory"

# Files
$ReportOutput = $FolderPath + $FolderName + "\" + $FolderName + "_report.csv"
$LogsOutput = $FolderPath + $FolderName + "\" + $FolderName + "_Logs.txt"

Add-ScriptLog -Color Cyan -Msg "Report will be generated at $($ReportOutput)"



#################################################################
# SCRIPT LOGIC
#################################################################

function Remove-SiteUser {
    param (
        $Site
    )

    Connect-PnPOnline -Url $Site.Url -Interactive -ErrorAction Stop
    
    $collUsers = Get-PnPUser | Where-Object { $_.Email -eq $UserToDelete }

    foreach($oUser in $collUsers) {
        #Remove-PnPUser $oUser.id -Force
        Add-ScriptLog -Color Green -Msg "User $($collUsers.Email) removed from site"
        Add-ReportRecord -Site $Site -User $oUser
    }

}

try {
    Connect-PnPOnline -Url $SPOAdminURL -Interactive -ErrorAction Stop
    Add-ScriptLog -Color Cyan -Msg "Connected to SharePoint Admin Center"

    $collSiteCollections = Get-PnPTenantSite -ErrorAction Stop | Where-Object{ ($_.Title -notlike "" -and $_.Template -notlike "*Redirect*") }
    Add-ScriptLog -Color Cyan -Msg "Collected Site Collections"
    Add-ScriptLog -Color Cyan -Msg "Number of SiteCollections: $($collSiteCollections.count)"
}
catch {
    Add-ScriptLog -Color Red -Msg "Error: $($_.Exception.Message)"
    break
}

$ItemCounter = 0
$ItemCounterStep = 1 / $collSiteCollections.Count
foreach($oSiteCollection in $collSiteCollections) {
    
    $PercentComplete = [math]::Round($ItemCounter/$collSiteCollections.Count * 100, 2)
    Add-ScriptLog -Color Yellow -Msg "$($PercentComplete)% Completed - Processing Site Collection: $($oSiteCollection.Title)"
    $ItemCounter++
    
    try {
        Set-PnPTenantSite -Url $oSiteCollection.Url -Owners $AdminUPN

        Remove-SiteUser -Site $oSiteCollection -ErrorAction Stop

        try {
            $collSubsites = Get-PnPSubWeb -Recurse -Includes HasUniqueRoleAssignments

            ForEach($oSubsite in $collSubsites) {

                $PercentComplete = [math]::Round( $PercentComplete + ( ($ItemCounterStep / $collSubsites.Count) * 100 ), 2 )
                Add-ScriptLog -Color Yellow -Msg "$($PercentComplete)% Completed - Processing Subsite: $($oSubsite.Title)"
            
                Remove-SiteUser -Site $oSubsite -ErrorAction Stop
            }

        }
        catch {
            Add-ScriptLog -Color Red -Msg "Error processing Subsite: $($oSiteCollection.Url)"
            Add-ScriptLog -Color Red -Msg "Error message: $($_.Exception.Message)"
        }

    }
    catch {
        Add-ScriptLog -Color Red -Msg "Error processing Site Collection: $($oSiteCollection.Url)"
        Add-ScriptLog -Color Red -Msg "Error message: $($_.Exception.Message)"
    }

}
$PercentComplete = [math]::Round($ItemCounter/$collSiteCollections.Count * 100, 1)
Add-ScriptLog -Color Cyan -Msg "$($PercentComplete)% Completed - Finished running script"
Add-ScriptLog -Color Cyan -Msg "Report generated at at $($ReportOutput)"
```
