#QuickFix #ErrorMessage #OneDrive #SharePointOnline #PUID #Permissions #PnP

<br>

```powershell
#################################################################
# DEFINE PARAMETERS FOR THE CASE
#################################################################
$AdminSiteURL = "https://<DOMAIN>-admin.sharepoint.com" # SharePoint Admin Center Url
$SiteCollAdmin = "<ADMIN@EMAIL.com>" # Global or SharePoint Admin used for loging running the script.
$AffectedUser = "<AFFECTEDUSER@EMAIL.com>" # Email of the affected user.



#################################################################
# REPORT AND LOGS FUNCTIONS
#################################################################

Function Add-ReportRecord($SiteURL, $Action)
{
    $Record = New-Object PSObject -Property ([ordered]@{
        "Site URL"          = $SiteURL
        "Action"            = $Action
        })
    
    $Record | Export-Csv -Path $ReportOutput -NoTypeInformation -Append
}

Function Add-ScriptLog($Color, $Msg)
{
    Write-host -f $Color $Msg
    $Date = Get-Date -Format "yyyy/MM/dd HH:mm"
    $Msg = $Date + " - " + $Msg
    Add-Content -Path $LogsOutput -Value $Msg
}

# Create Report location
$FolderPath = "$Env:USERPROFILE\Documents\"
$Date = Get-Date -Format "yyyyMMddHHmmss"
$ReportName = "IDMismatchSPO"
$FolderName = $Date + "_" + $ReportName
New-Item -Path $FolderPath -Name $FolderName -ItemType "directory"

# Files
$ReportOutput = $FolderPath + $FolderName + "\" + $FolderName + "_report.csv"
$LogsOutput = $FolderPath + $FolderName + "\" + $FolderName + "_Logs.txt"

Add-ScriptLog -Color Cyan -Msg "Report will be generated at $($ReportOutput)"



#################################################################
# SCRIPT LOGIC
#################################################################
function Remove-UserIDMismatch ($Site) {
    try {
        Connect-PnPOnline -Url $Site.Url -Interactive -ErrorAction Stop
        Add-ScriptLog -Color White -Msg "Connected to Site"

        $User = Get-PnPUser -Identity $properties.AccountName | Where-Object { $_.Email -eq $AffectedUser -and $_.UserId.NameId -ne $UserID }
        
        If ($User.Length -eq 0) {
            Add-ScriptLog -Color White -Msg "No issue found."
        }
        Else {
            Add-ScriptLog -Color White -Msg "User with incorrect SharePoint ID $($Site.UserId.NameId) found on this site."
            
            if($User.IsSiteAdmin) {
                # Remove-PnPSiteCollectionAdmin -Owners $AffectedUser -ErrorAction Stop
                Add-ScriptLog -Color White -Msg "User removed as Site Collection Admin."
                Add-ReportRecord -SiteURL $Site.Url -Action "Removed user as Site Collection Admin"
            }

            # Remove-PnPUser -Identity $User.ID -Force -ErrorAction Stop
            Add-ScriptLog -Color White -Msg "User removed from target Site"
            Add-ReportRecord -SiteURL $Site.Url -Action "Removed user from Site"
        }
    }
    catch {
        throw
    }
}


try {
    Connect-PnPOnline -Url $AdminSiteURL -Interactive -ErrorAction Stop
    Add-ScriptLog -Color Cyan -Msg "Connected to SharePoint Admin Center"

    $collSiteCollections = Get-PnPTenantSite | Where-Object{ ($_.Title -notlike "" -and $_.Template -notlike "*Redirect*") }
    #$collSiteCollections = Get-PnPTenantSite -IncludeOneDriveSites -Filter "Url -like '-my.sharepoint.com/personal/'" | Where-Object{ $_.Title -notlike "" -and $_.Template -notlike "*Redirect*" }
    Add-ScriptLog -Color Cyan -Msg "Collected all Site Collections: $($collSiteCollections.Count)"
}
catch {
    Add-ScriptLog -Color Red -Msg "Error: $($_.Exception.Message)"
    break
}


try {
    
    $properties = Get-PnPUserProfileProperty -Account $AffectedUser
    $UserID = $properties.UserProfileProperties.SID -replace ("i:0h.f|membership|", '')
    $UserID = $UserID -replace ('@live.com', '')
    $UserID = $UserID.Trim('|')

    Add-ScriptLog -Color Cyan -Msg "User Account name: $($properties.AccountName)"
    Add-ScriptLog -Color Cyan -Msg "User correct ID: $($UserID)"
    
}
catch {
    Add-ScriptLog -Color Red -Msg "Error message: '$($_.Exception.Message)'"
    Add-ScriptLog -Color Red -Msg "Error trace: '$($_.Exception.ScriptStackTrace)'"
    break
}


$ItemCounter = 0
ForEach($oSite in $collSiteCollections) {

    $PercentComplete = [math]::Round($ItemCounter/$collSiteCollections.Count * 100, 2)
    Add-ScriptLog -Color Yellow -Msg "$($PercentComplete)% Completed - Processing Site Collection: $($oSite.Url)"
    $ItemCounter++

    
    Try {
        Set-PnPTenantSite -Url $oSite.Url -Owners $SiteCollAdmin

        Remove-UserIDMismatch -Site $oSite -ErrorAction Stop
        
    }
    Catch {
        Add-ScriptLog -Color Red -Msg "Error while processing Site Collection '$($Site.Url)'"
        Add-ScriptLog -Color Red -Msg "Error message: '$($_.Exception.Message)'"
        Add-ScriptLog -Color Red -Msg "Error trace: '$($_.Exception.ScriptStackTrace)'"
        Add-ReportRecord -SiteURL $oSite.Url -Action $_.Exception.Message
    }

    Connect-PnPOnline -Url $oSite.Url -Interactive
    Remove-PnPSiteCollectionAdmin -Owners $SiteCollAdmin

}

if($collSiteCollections.Count -ne 0) { 
    $PercentComplete = [math]::Round($ItemCounter/$collSiteCollections.Count * 100, 1) 
    Add-ScriptLog -Color Cyan -Msg "$($PercentComplete)% Completed - Finished running script"
}
Add-ScriptLog -Color Cyan -Msg "Report generated at at $($ReportOutput)"
```
