#Report #SharePointOnline #OneDrive #PowerShell #PnP #DocumentLibrary #ItemList #SharedLink #RoleAssigments 

<br>

## User in Site all people list

```powershell
#################################################################
# DEFINE PARAMETERS FOR THE CASE
#################################################################
$SPOAdminURL = "https://Domain-admin.sharepoint.com"
$AdminUPN = "admin@email.com"
$UserEmail = "user@email.com"



#################################################################
# REPORT AND LOGS FUNCTIONS
#################################################################
function Add-ReportRecord {
    param (
        $SiteUrl,
        $Remarks
    )

    $Record = New-Object PSObject -Property ([ordered]@{
        SiteURL = $SiteURL
        Remarks = $Remarks
        })
    
    $Record | Export-Csv -Path $ReportOutput -NoTypeInformation -Append
}

function Add-ReportRecord {
    param (
        $SiteUrl,
        $List,
        $Remarks
    )

    $Record = New-Object PSObject -Property ([ordered]@{
        SiteURL = $SiteURL
        Title = $List.Title
        ListType = $List.BaseType
        ListDefaultViewUrl = $List.DefaultViewUrl
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

# Create Report location
$FolderPath = "$Env:USERPROFILE\Documents\"
$Date = Get-Date -Format "yyyyMMddHHmmss"
$ReportName = "UserAccess"
$FolderName = $Date + "_" + $ReportName
New-Item -Path $FolderPath -Name $FolderName -ItemType "directory"

# Files
$ReportOutput = $FolderPath + $FolderName + "\" + $FolderName + "_report.csv"
$LogsOutput = $FolderPath + $FolderName + "\" + $FolderName + "_Logs.txt"

Add-ScriptLog -Color Cyan -Msg "Report will be generated at $($ReportOutput)"



#################################################################
# SCRIPT LOGIC
#################################################################

function Get-User {
    param (
        $SiteUrl
    )

    Connect-PnPOnline -Url $SiteUrl -Interactive -ErrorAction Stop
    
    $oUser = Get-PnPUser -WithRightsAssigned | Where-Object { $_.Email -eq $UserEmail }

    If ($oUser) {
        Add-ReportRecord -SiteUrl $SiteURL -Remarks "User found"
    }
}


try {
    Connect-PnPOnline -Url $SPOAdminURL -Interactive -ErrorAction Stop
    Add-ScriptLog -Color Cyan -Msg "Connected to SharePoint Admin Center"

    $collSiteCollections = Get-PnPTenantSite -IncludeOneDriveSites | Where-Object { $_.Title -notlike "" -and $_.Template -notlike "*Redirect*" }
    Add-ScriptLog -Color Cyan -Msg "Collected Site Collections: $($collSiteCollections.count)"
}
catch {
    Add-ScriptLog -Color Red -Msg "Error: $($_.Exception.Message)"
    break
}

$ItemCounter = 0
$ItemCounterStep = 1 / $collSiteCollections.Count
foreach($oSiteCollection in $collSiteCollections){
    
    $PercentComplete = [math]::Round($ItemCounter/$collSiteCollections.Count * 100, 2)
    Add-ScriptLog -Color Yellow -Msg "$($PercentComplete)% Completed - Processing Site Collection: $($oSiteCollection.Url)"
    $ItemCounter++
    
    try {
        Set-PnPTenantSite -Url $oSiteCollection.Url -Owners $AdminUPN

        Get-User -Site $oSiteCollection.Url

        $collSubsites = Get-PnPSubWeb -Recurse

        ForEach($oSubsite in $collSubsites) {

            $PercentComplete = [math]::Round( $PercentComplete + ( ($ItemCounterStep / $collSubsites.Count) * 100 ), 2 )
            Add-ScriptLog -Color Yellow -Msg "$($PercentComplete)% Completed - Processing Subsite: $($oSubsite.Title)"
        
            try {
                Get-User -Site $oSubsite.Url
            }
            catch {
                Add-ScriptLog -Color Red -Msg "Error while processing '$($oSubsite.Url)"
                Add-ScriptLog -Color Red -Msg "Error message: '$($_.Exception.Message)'"
                Add-ScriptLog -Color Red -Msg "Error trace: '$($_InvocationInfo.ScriptLineNumber)'"
                Add-ReportRecord -SiteUrl $oSubsite.Url -Remarks $_.Exception.Message
            }
        }

    }
    catch{
        Add-ScriptLog -Color Red -Msg "Error while processing '$($oSiteCollection.Url)"
        Add-ScriptLog -Color Red -Msg "Error message: '$($_.Exception.Message)'"
        Add-ScriptLog -Color Red -Msg "Error trace: '$($_.InvocationInfo.ScriptLineNumber)'"
        Add-ReportRecord -SiteUrl $oSiteCollection.Url -Remarks $_.Exception.Message
    }
    Connect-PnPOnline -Url $oSiteCollection.Url -Interactive
    Remove-PnPSiteCollectionAdmin -Owners $SiteCollAdmin
}
Add-ScriptLog -Color Cyan -Msg "100% Completed - Finished running script"
Add-ScriptLog -Color Cyan -Msg "Report generated at at $($ReportOutput)"
```