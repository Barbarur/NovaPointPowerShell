#Automation #PnP #PowerShell #SharePointOnline #SiteCollection #OneDrive #SiteCollectionAdmin #AzureAD 

<br>

## Add Users as Admin to their own OneDrive

```powershell
#################################################################
# DEFINE PARAMETERS FOR THE CASE
#################################################################
$SiteAdminURL = "https://<Domain>-admin.sharepoint.com/" # SharePoint Admin Center URL



#################################################################
# REPORT AND LOGS FUNCTIONS
#################################################################

Function Add-ReportRecord {
    param (
        $UserUPN,
        $PersonalUrl,
        $Status
    )

    $Record = New-Object PSObject -Property ([ordered]@{
        UserUPN = $UserUPN
        PersonalUrl = $PersonalUrl
        Status = $Status
        })
    
    $Record | Export-Csv -Path $ReportOutput -NoTypeInformation -Append
}

Function Add-ScriptLog($Color, $Msg) {
    Write-host -f $Color $Msg
    $Date = Get-Date -Format "yy/MM/dd HH:mm"
    $Msg = $Date + " - " + $Msg
    Add-Content -Path $LogsOutput -Value $Msg
}

# Create Report location
$FolderPath = "$Env:USERPROFILE\Documents\SPOSolutions\"
$Date = Get-Date -Format "yyyyMMddHHmmss"
$ReportName = "AddOneDriveUserAdmin"
$FolderName = $Date + "_" + $ReportName
New-Item -Path $FolderPath -Name $FolderName -ItemType "directory"

# Files
$ReportOutput = $FolderPath + $FolderName + "\" + $FolderName + "_report.csv"
$LogsOutput = $FolderPath + $FolderName + "\" + $FolderName + "_Logs.txt"

Add-ScriptLog -Color Cyan -Msg "Report will be generated at $($ReportOutput)"



#################################################################
# SCRIPT LOGIC
#################################################################

try {

    Connect-PnPOnline -Url $SiteAdminURL -Interactive -ErrorAction Stop
    Add-ScriptLog -Color Cyan -Msg "Connected to SharePoint Admin Center"

    Connect-AzureAD -ErrorAction Stop
    Add-ScriptLog -Color Cyan -Msg "Connected to AAD"

    $collUsers = Get-AzureADUser -All $true
    Add-ScriptLog -Color Cyan -Msg "Users Collected: $($collUsers.Count)"
}
catch {

    Add-ScriptLog -Color Red -Msg "Error message: '$($_.Exception.Message)'"
    Add-ScriptLog -Color Red -Msg "Error trace: '$($_.Exception.ScriptStackTrace)'"
    Disconnect-ExchangeOnline
    break

}

$ItemCounter = 0
foreach($oUser in $collUsers) {

    $PercentComplete = [math]::Round($ItemCounter/$collUsers.Count * 100, 2)
    Add-ScriptLog -Color Yellow -Msg "$($PercentComplete)% Completed - Processing User '$($oUser.UserPrincipalName)'"
    $ItemCounter++
    
    Try{
        $userProperties = Get-PnPUserProfileProperty -Account $oUser.UserPrincipalName

        Set-PnPTenantSite -Url $userProperties.PersonalUrl -Owners $oUser.UserPrincipalName
        Add-ReportRecord -UserUPN $oUser.UserPrincipalName -PersonalUrl $userProperties.PersonalUrl -Status "User added as Admin Correctly"
    }
    catch{
        Add-ScriptLog -Color Red -Msg "Error while processing User '$($oUser.UserPrincipalName)'"
        Add-ScriptLog -Color Red -Msg "Error message: '$($_.Exception.Message)'"
        Add-ScriptLog -Color Red -Msg "Error trace: '$($_.InvocationInfo.ScriptLineNumber)'"
        Add-ReportRecord -UserUPN $oUser.UserPrincipalName -PersonalUrl $userProperties.PersonalUrl -Status $_.Exception.Message
    }
}

if($collUsers.Count -ne 0) {

    $PercentComplete = [math]::Round( $ItemCounter/$collUsers.Count * 100, 2 )
    Add-ScriptLog -Color Cyan -Msg "$($PercentComplete)% Completed - Finished running script"

}
Add-ScriptLog -Color Cyan -Msg "Report generated at at $($ReportOutput)"

```