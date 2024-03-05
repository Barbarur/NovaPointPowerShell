#Automation #PowerShell #PnP #DocumentLibrary #SiteCollection #Search 

<br>

## List and Libraries for specific Site Collections

```powershell
################################################################
# PARAMETERS TO BE CHANGED TO MATCH CURRENT CASE
################################################################
$AdminSiteURL = "https://<Domain>-admin.sharepoint.com"
$SiteCollAdmin = "<admin@email.com>" 
$CSVFile = "" # Local path to the CSV file



################################################################
# REPORT AND LOGS FUNCTIONS
################################################################

function Add-ReportRecord {
    param (
        $SiteUrl,
        $ListID,
        $ListTitle,
        $Remarks
    )

    $Record = New-Object PSObject -Property ([ordered]@{
        SiteUrl = $SiteUrl
        ListID = $Owners
        ListTitle = $Members
        Remarks = $Remarks
        })

    $Record | Export-Csv -Path $ReportOutput -NoTypeInformation -Append
}

Function Add-ScriptLog($Color, $Msg) {
    $Date = Get-Date -Format "yyyy/MM/dd HH:mm"
    $Msg = $Date + " - " + $Msg
    Add-Content -Path $LogsOutput -Value $Msg
    Write-host -f $Color $Msg
}


# Create Report location
$FolderPath = "$Env:USERPROFILE\Documents\"
$Date = Get-Date -Format "yyyyMMddHHmmss"
$ReportName = "RemoveVisitors"
$FolderName = $Date + "_" + $ReportName
New-Item -Path $FolderPath -Name $FolderName -ItemType "directory"

# Files
$ReportOutput = $FolderPath + $FolderName + "\" + $FolderName + "_report.csv"
$LogsOutput = $FolderPath + $FolderName + "\" + $FolderName + "_Logs.txt"

Add-ScriptLog -Color Cyan -Msg "Report will be generated at $($ReportOutput)"



#################################################################
# SCRIPT LOGIC
#################################################################

function Find-List {
    param (
        $SiteUrl,
        $ListID
    )
    
    Add-ScriptLog -Color Yellow -Msg "Finding list '$($ListID)'"

    Connect-PnPOnline -Url $SiteUrl -Interactive 

    $List = Get-PnPList -Identity $ListID -Includes RoleAssignments
    
    $List.BreakRoleInheritance($true, $false)
    Add-ScriptLog -Color Yellow -Msg "Broken permissions inheritance for the list"

    $ctx = Get-PnPContext
    $AssociatedVisitorGroup = Get-PnPGroup -AssociatedVisitorGroup
    $List.RoleAssignments.Groups.RemoveByLoginName("$($AssociatedVisitorGroup.Title)")
    $ctx.ExecuteQuery()
    Add-ScriptLog -Color Yellow -Msg "Removed visitors group '$($AssociatedVisitorGroup.Title)'"

    Add-ReportRecord -SiteUrl $oSite.Url -ListID $ListID -ListTitle $List.Title -Remarks "Search disabled for this list"
}

try {
    $collSiteCollections = Import-CSV $CSVFile
    Add-ScriptLog -Color Cyan -Msg "Collected Site Collections: $($collSiteCollections.count)"
}
catch {
    Add-ScriptLog -Color Red -Msg "Error: $($_.Exception.Message)"
    break
}


$ItemCounter = 0
ForEach($oSite in $collSiteCollections) {

    $PercentComplete = [math]::Round($ItemCounter/$collSiteCollections.Count * 100, 2)
    Add-ScriptLog -Color Cyan -Msg "$($PercentComplete)% Completed"
    Add-ScriptLog -Color Yellow -Msg "Processing Site Collection: $($oSite.URL)"
    $ItemCounter++

    Try {
        Connect-PnPOnline -Url $AdminSiteURL -Interactive -ErrorAction Stop
        Set-PnPTenantSite -Url $oSite.Url -Owners $SiteCollAdmin -ErrorAction Stop
        Add-ScriptLog -Color Yellow -Msg "Added Site Collection Admin"

        Find-List -SiteUrl $oSite.Url -ListID $oSite.ListID
    }
    Catch {
        Add-ScriptLog -Color Red -Msg "Error while processing Site Collection '$($oSite.Url)'"
        Add-ScriptLog -Color Red -Msg "Error message: '$($_.Exception.Message)'"
        Add-ScriptLog -Color Red -Msg "Error Script Line: '$($_.InvocationInfo.ScriptLineNumber)'"
        Add-ReportRecord -SiteUrl $oSite.Url -Remarks $_.Exception.Message
    }

    Connect-PnPOnline -Url $AdminSiteURL -Interactive
    Remove-PnPSiteCollectionAdmin -Owners $SiteCollAdmin
}

Add-ScriptLog -Color Cyan -Msg "100% Completed - Finished running script"
Add-ScriptLog -Color Cyan -Msg "Report generated at at $($ReportOutput)"
```