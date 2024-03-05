#Report #SharePointOnline #PowerShell #PnP #GuestUser #ExternalUser

<br>

## External users in all Sites

```powershell
#################################################################
# DEFINE PARAMETERS FOR THE CASE
#################################################################
$AdminSiteURL = "https://{Domain}-admin.sharepoint.com"
$SiteCollAdmin = "admin@emai.com"


#################################################################
# REPORT AND LOGS FUNCTIONS
#################################################################

function Add-ReportRecord {
    param (
        $SiteUrl,
        $ExtUsers,
        $Remarks
    )

    $Record = New-Object PSObject -Property ([ordered]@{
        SiteUrl = $SiteUrl
        ExtUsers = $ExtUsers
        Remarks = $Remarks
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

$FolderPath = "$Env:USERPROFILE\Documents\SPOSolutions\"
$Date = Get-Date -Format "yyyyMMddHHmmss"
$ReportName = "GuestReport"
$FolderName = $Date + "_" + $ReportName
New-Item -Path $FolderPath -Name $FolderName -ItemType "directory"

$ReportOutput = $FolderPath + $FolderName + "\" + $FolderName + "_report.csv"
$LogsOutput = $FolderPath + $FolderName + "\" + $FolderName + "_Logs.txt"

Add-ScriptLog -Color Cyan -Msg "Report will be generated at $($ReportOutput)"



#################################################################
# SCRIPT LOGIC
#################################################################

try {
    Connect-SPOService -Url $AdminSiteURL -ErrorAction Stop
    Add-ScriptLog -Color Cyan -Msg "Connected to SharePoint Admin Center"
    
    $collSiteCollections = Get-SPOSite -Limit ALL -IncludePersonalSite $False -ErrorAction Stop | Where-Object{ $_.Title -notlike "" -and $_.Template -notlike "*Redirect*" }
    Add-ScriptLog -Color Cyan -Msg "Collected Site Collections: $($collSiteCollections.count)"  
}
catch {
    Add-ScriptLog -Color Red -Msg "Error: $($_.Exception.Message)"
    break
}

$ItemCounter = 0
ForEach($oSite in $collSiteCollections) {

    $PercentComplete = [math]::Round($ItemCounter/$collSiteCollections.Count * 100, 2)
    Add-ScriptLog -Color Yellow -Msg "$($PercentComplete)% Completed - Processing Site Collection: $($oSite.URL)"
    $ItemCounter++

    Try {
        Set-SPOUser -Site $oSite.Url -LoginName $SiteCollAdmin -IsSiteCollectionAdmin $True -ErrorAction Stop

        $collExtUsers = Get-SPOUser -Site $oSite.Url -Limit ALL | Where-Object {$_.UserType -eq "Guest"}
        $ExtUsersLoginName = ""
        foreach($oUser in $collExtUsers) {
            $ExtUsersLoginName += "$($oUser.LoginName)"
        }
        Add-ReportRecord -SiteUrl $oSite.Url -ExtUsers $ExtUsersLoginName

    }
    Catch {
        Add-ScriptLog -Color Red -Msg "Error while processing Site '$($oSite.Url)'"
        Add-ScriptLog -Color Red -Msg "Error message: '$($_.Exception.Message)'"
        Add-ScriptLog -Color Red -Msg "Error Script Line: '$($_.InvocationInfo.ScriptLineNumber)'"
        Add-ReportRecord -SiteUrl $oSite.Url -Remarks $_.Exception.Message
    }

    Set-SPOUser -Site $oSite.Url -LoginName $SiteCollAdmin -IsSiteCollectionAdmin $True
}

if($collSiteCollections.Count -ne 0) { 
    $PercentComplete = [math]::Round($ItemCounter/$collSiteCollections.Count * 100, 1) 
    Add-ScriptLog -Color Cyan -Msg "$($PercentComplete)% Completed - Finished running script"
}
Add-ScriptLog -Color Cyan -Msg "Report generated at at $($ReportOutput)"


```