#Report #SharePointOnline #Onedrive #PowerShell #PnP #SiteCollection #Subsite 

<br>

```powershell
#################################################################
# DEFINE PARAMETERS FOR THE CASE
#################################################################
$SiteURL= "https://<DOMAIN>.sharepoint.com/sites/<SITENAME>"
$ReportName = "FolderPathReport"



#################################################################
# REPORT AND LOGS FUNCTIONS
#################################################################
# Add new record on the report
Function Add-ReportRecord() {
    $Record = New-Object PSObject -Property ([ordered]@{
        "List Name"     = $ListName
        "List URL"      = $ListRelativeURL
        "Item ID"       = $ItemID
        "Item Name"     = $ItemName
        "Item URL"      = $ItemRelativeURL
        })
    
    $Record | Export-Csv -Path $ReportOutput -NoTypeInformation -Append
}

# Add Log of the Script
Function Add-ScriptLog($Color, $Msg) {
    $Date = Get-Date -Format "yyyy/MM/dd HH:mm K"
    $Msg = $Date + " - " + $Msg
    Add-Content -Path $LogsOutput -Value $Msg
    Write-host -f $Color $Msg
}

# Create Report location
$Date = Get-Date -Format FileDateTime
$FolderName = $Date + "_" + $ReportName
$FolderPath = "$Env:USERPROFILE\Documents\"
New-Item -Path $FolderPath -Name $FolderName -ItemType "directory"
$ReportOutput = $FolderPath + $FolderName + "\" + $ReportName + ".csv"

# Create logs file
$LogsName = $ReportName + "_Logs.txt"
$LogsOutput = $FolderPath + $FolderName + "\" + $LogsName

Add-ScriptLog -Color Cyan -Msg "Report will be generated at $($ReportOutput)"



#################################################################
# SCRIPT LOGIC
#################################################################
Try {
    Connect-PnPOnline -Url $SiteURL -Interactive -ErrorAction Stop
    $ExcludedListsList = @("appdata", "appfiles", "Composed Looks", "Converted Forms", "Form Templates", "List Template Gallery", "Master Page Gallery", "Preservation Hold Library", "Project Policy Item List", "Site Assets", "Site Pages", "Solution Gallery", "Style Library", "TaxonomyHiddenList", "Theme Gallery", "User Information List", "Web Part Gallery")
    $ListsList = Get-PnPList | Where-Object {$_.Hidden -eq $False -and $_.Title -notin $ExcludedListsList -and $_.BaseType -eq "DocumentLibrary"}
    Add-ScriptLog -Color Cyan -Msg "Connected to Site and Collected Items"
}
Catch {
    Add-ScriptLog -Color Red -Msg "Error: $($_.Exception.Message)"
}

# Iterate through the Lists
ForEach ($List in $ListsList) {
    # Report List information
    $ListName               = $List.Title
    $ListRelativeURL        = $List.DefaultViewUrl -Replace ('Forms/AllItems.aspx','') -Replace ('AllItems.aspx','')

    # Process notification parameters
    $ItemCounter = 0
    Add-ScriptLog -Color Yellow -Msg "Collecting items on $($List.BaseType) '$($List.Title)'"
    
    # Get all folders and iterate
    $ListItems = Get-PnPListItem -List $List.Title -PageSize 2000 | Where-Object { $_.FileSystemObjectType -eq "Folder" }
    ForEach($Item in $ListItems) {
        # Adding notification and logs
        $PercentComplete = [math]::Round($ItemCounter/$Items.Count*100,1)
        Add-ScriptLog -Color Yellow -Msg "$($PercentComplete)% Completed of $($List.BaseType) '$($List.Title)' - Adding record $($Item["FileRef"])"
        $ItemCounter++

        # Report Item infomation
        $ItemID              = $Item.Id
        $ItemName            = $Item["FileLeafRef"]
        $ItemRelativeURL     = $Item["FileRef"] -Replace ($ListRelativeURL,'')

        Add-ReportRecord
    }
    $PercentComplete = [math]::Round($ItemCounter/$ItemsList.Count * 100, 1)
    Add-ScriptLog -Color Yellow -Msg "$($PercentComplete)% Completed - Adding items from $($List.BaseType) '$($List.Title)'"
}
Add-ScriptLog -Color Cyan -Msg "Completed Running Script"
Add-ScriptLog -Color Cyan -Msg "Report generated at at $($ReportOutput)"
```