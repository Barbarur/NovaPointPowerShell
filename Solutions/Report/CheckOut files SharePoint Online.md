#Report #PnP #PowerShell #SharePointOnline #SiteCollection #DocumentLibrary #CheckOut

<br>

## Check-Out files in a document library of a site

```powershell
#################################################################
# DEFINE PARAMETERS FOR THE CASE
#################################################################
$SiteURL = "https://<Domain>-admin.sharepoint.com/sites/<SiteName>/"
$ListName = "LibraryName"



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
        CheckOutBy = $Item.FieldValues.CheckoutUser.Email
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
$ReportName = "CheckOutReport"
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

    $oList = Get-PnPList $ListName
    Add-ScriptLog -Color White -Msg "Collected List"

    if($oList.BaseType -ne "DocumentLibrary") {
        Add-ScriptLog -Color Red -Msg "This is not a Document Library"
        return
    }

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



function Find-Items {
    param (
        $SiteUrl,
        $List
    )
    
    Add-ScriptLog -Color White -Msg "Finding '$($List.ItemCount)' Files and Items in '$($List.Title)'"

    $collItems = Get-PnPListItem -List $List.Title -PageSize 5000

    foreach ( $oItem in $collItems) {
        
        Add-ScriptLog -Color White -Msg "Processing '$($oItem["FileRef"])'"

        if ($oItem.FieldValues.CheckoutUser.Email) {

            Add-ReportRecord -SiteUrl $SiteUrl -ListTitle $List.Title -Item $oItem
        
        }
    }
}


try {
    Connect-PnPOnline -Url $SiteURL -Interactive -ErrorAction Stop
    Add-ScriptLog -Color Cyan -Msg "Connected to SharePoint Site"
    
    Try {
        Find-Lists -SiteUrl $SiteURL
    }
    Catch {
        Add-ScriptLog -Color Red -Msg "Error while processing Site '$($oSite.Url)'"
        Add-ScriptLog -Color Red -Msg "Error message: '$($_.Exception.Message)'"
        Add-ScriptLog -Color Red -Msg "Error Script Line: '$($_.InvocationInfo.ScriptLineNumber)'"
        Add-ReportRecord -SiteUrl $SiteURL -Remarks $_.Exception.Message
    }
}
catch {
    Add-ScriptLog -Color Red -Msg "Error: $($_.Exception.Message)"
    break
}

if($collSiteCollections.Count -ne 0) { 
    $PercentComplete = [math]::Round($ItemCounter/$collLists.Count * 100, 1) 
    Add-ScriptLog -Color Cyan -Msg "$($PercentComplete)% Completed - Finished running script"
}
Add-ScriptLog -Color Cyan -Msg "Report generated at at $($ReportOutput)"


```
