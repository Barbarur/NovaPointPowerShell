#Report #SharePointOnline #OneDrive #PowerShell #PnP #DocumentLibrary

<br>

```powershell
#################################################################
# DEFINE PARAMETERS FOR THE CASE
#################################################################
$AdminSiteURL = "https://Domain-admin.sharepoint.com"
$SiteCollAdmin = "admin@email.com" 
$FileExtension = "*.mp4"



#################################################################
# REPORT AND LOGS FUNCTIONS
#################################################################

function Add-ReportRecord {
    param (
        $SiteUrl,
        $ListTitle,
        $Item,
        $Remarks
    )

    $Record = New-Object PSObject -Property ([ordered]@{
        SiteUrl = $SiteUrl
        ListTitle = $ListTitle
        ID = $Item.Id
        Name = $Item["FileLeafRef"]
        Type = $Item.FileSystemObjectType
        Extension = $Item["File_x0020_Type"]
        ItemURL = $Item["FileRef"]
        Created = $Item["Created"]
        Createdby = $Item["Author"].Email
        Modified = $Item["Modified"]
        Modifiedby = $Item["Editor"].Email
        VersionNo = $Item["_UIVersionString"]
        ItemSizeMB = [Math]::Round(($Item["File_x0020_Size"]/1MB),1)
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
$FolderPath = "$Env:USERPROFILE\Documents\SPOSolutions\"
$Date = Get-Date -Format "yyyyMMddHHmmss"
$ReportName = "ItemReport"
$FolderName = $Date + "_" + $ReportName
New-Item -Path $FolderPath -Name $FolderName -ItemType "directory"

# Files
$ReportOutput = $FolderPath + $FolderName + "\" + $FolderName + "_report.csv"
$LogsOutput = $FolderPath + $FolderName + "\" + $FolderName + "_Logs.txt"

Add-ScriptLog -Color Cyan -Msg "Report will be generated at $($ReportOutput)"



#################################################################
# SCRIPT LOGIC
#################################################################

function Find-Lists {
    param (
        $SiteUrl
    )
    
    Add-ScriptLog -Color White -Msg "Finding Libraries and Lists in Site '$($SiteUrl)'"

    $collLists = Get-PnPList | Where-Object { $_.Hidden -eq $False -and $_.BaseType -eq "DocumentLibrary" }
    Add-ScriptLog -Color White -Msg "Collected all Lists: $($collLists.Count)"
    
    ForEach($oList in $collLists) {

        Try {
            
            Find-Items -SiteUrl $SiteUrl -List $oList

        }
        Catch {

            Add-ScriptLog -Color Red -Msg "Error while processing List '$($oList.Title)'"
            Add-ScriptLog -Color Red -Msg "Error message: '$($_.Exception.Message)'"
            Add-ScriptLog -Color Red -Msg "Error trace: '$($_.Exception.ScriptStackTrace)'"
            Add-ReportRecord -SiteUrl $SiteUrl -ListTitle $oList.Title -Remarks $_.Exception.Message
        
        }
    }
}

function Find-Items {
    param (
        $SiteUrl,
        $List
    )
    
    Add-ScriptLog -Color White -Msg "Finding '$($List.ItemCount)' Files and Items in '$($List.Title)'"

    $collItems = Get-PnPListItem -List $List.Title -PageSize 5000 | Where-Object { $_["FileLeafRef"] -like $FileExtension }

    foreach ( $oItem in $collItems) {        
        Add-ScriptLog -Color White -Msg "Processing '$($oItem["FileRef"])'"

        Add-ReportRecord -SiteUrl $SiteUrl -ListTitle $List.Title -Item $oItem
    }
}


try {
    Connect-PnPOnline -Url $AdminSiteURL -Interactive -ErrorAction Stop
    Add-ScriptLog -Color Cyan -Msg "Connected to SharePoint Admin Center"
    
    if($CSVFile) {
        $collSiteCollections = Import-CSV $CSVFile
    }
    else {
        $collSiteCollections = Get-PnPTenantSite -ErrorAction Stop | Where-Object{ ($_.Title -notlike "" -and $_.Template -notlike "*Redirect*") }
    }
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
        Set-PnPTenantSite -Url $oSite.Url -Owners $SiteCollAdmin -ErrorAction Stop

        Connect-PnPOnline -Url $oSite.Url -Interactive

        Find-Lists -SiteUrl $oSite.Url
    }
    Catch {
        Add-ScriptLog -Color Red -Msg "Error while processing Site '$($oSite.Url)'"
        Add-ScriptLog -Color Red -Msg "Error message: '$($_.Exception.Message)'"
        Add-ScriptLog -Color Red -Msg "Error Script Line: '$($_.InvocationInfo.ScriptLineNumber)'"
        Add-ReportRecord -SiteUrl $oSite.Url -Remarks $_.Exception.Message
    }

    Connect-PnPOnline -Url $oSite.Url -Interactive
    Remove-PnPSiteCollectionAdmin -Owners $SiteCollAdmin
}

Add-ScriptLog -Color Cyan -Msg "100% Completed - Finished running script"
Add-ScriptLog -Color Cyan -Msg "Report generated at at $($ReportOutput)"
```