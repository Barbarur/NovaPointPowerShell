#Report #PowerShell #PnP #ItemList #DocumentLibrary #SiteCollection #Subsite 

<br>

## List and Libraries across all Site

```powershell
################################################################
# PARAMETERS TO BE CHANGED TO MATCH CURRENT CASE
################################################################
$AdminSiteURL = "https://Domain-admin.sharepoint.com"
$ClientId = "00000000-0000-0000-0000-000000000000"
$SiteCollAdmin = "admin@email.com"



################################################################
# REPORT AND LOGS FUNCTIONS
################################################################

function Add-ReportRecord {
    param (
        $SiteUrl,
        $SiteAdmin,
        $List,
        $Remarks
    )

    $Record = New-Object PSObject -Property ([ordered]@{
        SiteURL = $SiteURL
        SiteAdmin = $SiteAdmin
        Title = $List.Title
        ListType = $List.BaseType
        ListServerRelativeUrl = $List.RootFolder.ServerRelativeUrl
        ListLastModified = $List.LastItemUserModifiedDate
        Remarks = $Remarks
        })
    
    $Record | Export-Csv -Path $ReportOutput -NoTypeInformation -Append
}

Function Add-ScriptLog($Color, $Msg)
{
    $Date = Get-Date -Format "yyyy/MM/dd HH:mm"
    $Msg = $Date + " - " + $Msg
    Add-Content -Path $LogsOutput -Value $Msg
    Write-host -f $Color $Msg
}

$Date = Get-Date -Format "yyyyMMdd_HHmmss"
$ReportName = "ListReport"
$FolderName = $Date + "_" + $ReportName
$FolderPath = "$Env:USERPROFILE\Documents\"
New-Item -Path $FolderPath -Name $FolderName -ItemType "directory"
$ReportOutput = $FolderPath + $FolderName + "\" + $ReportName + ".csv"

$LogsName = $ReportName + "_Logs.txt"
$LogsOutput = $FolderPath + $FolderName + "\" + $LogsName

Add-ScriptLog -Color Cyan -Msg "Report will be generated at $($ReportOutput)"



#################################################################
# SCRIPT LOGIC
#################################################################
Function Find-SiteLists($SiteURL, $Admins)
{
    Connect-PnPOnline -Url $SiteURL -ClientId $ClientId -Interactive
    Add-ScriptLog -Color Yellow -Msg "Processing Site: $($SiteURL)"

    $collLists = Get-PnPList | Where-Object { $_.Hidden -eq $False }

    ForEach($oList in $collLists) {
        Add-ReportRecord -SiteUrl $SiteURL -SiteAdmin $Admins -List $oList
    }
}


try {
    Connect-PnPOnline -Url $AdminSiteURL -ClientId $ClientId -Interactive -ErrorAction Stop
    Add-ScriptLog -Color Cyan -Msg "Connected to SharePoint Admin Center"

    $collSiteCollections = Get-PnPTenantSite -Detailed -ErrorAction Stop | Where-Object{ $_.Title -notlike "" -and $_.Template -notlike "*Redirect*" }
    Add-ScriptLog -Color Cyan -Msg "Collected Site Collections: $($collSiteCollections.count)"
}
catch {
    Add-ScriptLog -Color Red -Msg "Error: $($_.Exception.Message)"
    break
}

$ItemCounter = 0
ForEach($oSite in $collSiteCollections) {

    $PercentComplete = [math]::Round($ItemCounter/$collSiteCollections.Count * 100, 2)
    Add-ScriptLog -Color Yellow -Msg "$($PercentComplete)% Completed"
    $ItemCounter++

    Try{
        Set-PnPTenantSite -Url $oSite.Url -Owners $SiteCollAdmin -ErrorAction Stop

        $collAdmins = Get-PnPSiteCollectionAdmin
        $Admins = ""
        foreach ($admin in $collAdmins) { $Admins += " $($admin.Email)"}
        #Add-ScriptLog -Color Green -Msg $Admins

        Find-SiteLists -SiteURL $oSite.Url -Admins $Admins

        $collSubSites = Get-PnPSubWeb -Recurse

        ForEach($oSubsite in $collSubSites) {
            Find-SiteLists -SiteURL $oSubsite.Url -Admins $Admins
        }
    }
    Catch{
        Add-ScriptLog -Color Red -Msg "Error while processing Item '$($oSite.Url)"
        Add-ScriptLog -Color Red -Msg "Error message: '$($_.Exception.Message)'"
        Add-ScriptLog -Color Red -Msg "Error trace: '$($_.InvocationInfo.ScriptLineNumber)'"
        Add-ReportRecord -SiteUrl $SiteURL -Remarks $_.Exception.Message
    }
}
Add-ScriptLog -Color Cyan -Msg "100% Completed - Finished running script"
Add-ScriptLog -Color Cyan -Msg "Report generated at at $($ReportOutput)"
```

<br>

## List and Libraries with IRM enabled across all Site

```powershell
################################################################
# PARAMETERS TO BE CHANGED TO MATCH CURRENT CASE
################################################################
$AdminSiteURL = "https://<DOMAIN>-admin.sharepoint.com"
$SiteCollAdmin = "<USER@EMAIL.com>"



################################################################
# REPORT AND LOGS FUNCTIONS
################################################################
# Add new record on the report
Function Add-ReportRecord($SiteURL, $LibraryName, $Remarks)
{
    $Record = New-Object PSObject -Property ([ordered]@{
        "Site URL"          = $SiteURL
        "Library Name"      = $LibraryName
        "Remarks"           = $Remarks
        })
    
    $Record | Export-Csv -Path $ReportOutput -NoTypeInformation -Append
}

# Add Log of the Script
Function Add-ScriptLog($Color, $Msg)
{
    $Date = Get-Date -Format "yyyy/MM/dd HH:mm"
    $Msg = $Date + " - " + $Msg
    Add-Content -Path $LogsOutput -Value $Msg
    Write-host -f $Color $Msg
}

# Create Report location
$Date = Get-Date -Format "yyyyMMdd_HHmmss"
$ReportName = "IRMLibraryReport"
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
Function Find-IRMLibray($SiteURL)
{
    Connect-PnPOnline -Url $SiteURL -Interactive

    $ExcludedListsList = @("appdata", "appfiles", "Composed Looks", "Converted Forms", "Form Templates", "List Template Gallery", "Master Page Gallery", "Preservation Hold Library", "Project Policy Item List", "Site Assets", "Site Pages", "Solution Gallery", "Style Library", "TaxonomyHiddenList", "Theme Gallery", "User Information List", "Web Part Gallery")
    $ListsList = Get-PnPList | Where-Object {$_.Hidden -eq $False -and $_.Title -notin $ExcludedListsList}

    ForEach($List in $ListsList) {
        If($list.IrmEnabled -eq $true) {
            Add-ScriptLog -Color Magenta -Msg "$($PercentComplete)% Completed - Checking Site: $($Site.Title) - IRM Enabled for $($List.BaseType) '$($List.Title)'"
            Add-ReportRecord -SiteURL $SiteURL -LibraryName $List.Title
        }
        Else {
            Add-ScriptLog -Color Yellow -Msg "$($PercentComplete)% Completed - Checking Site: $($Site.Title) - IRM NOT enabled for $($List.BaseType) '$($List.Title)'"
        }
    }
}

# Connect to SharePoint Site and collect subsites
try {
    Connect-PnPOnline -Url $AdminSiteURL -Interactive -ErrorAction Stop
    Add-ScriptLog -Color Cyan -Msg "Connected to SharePoint Admin Center"
    $SitesList = Get-PnPTenantSite -ErrorAction Stop | Where-Object{ ($_.Title -notlike "" -and $_.Template -notlike "*Redirect*") }
    Add-ScriptLog -Color Cyan -Msg "Collected Site Collections"
    Add-ScriptLog -Color Cyan -Msg "Number of SiteCollections: $($SitesList.count)"
}
catch {
    Add-ScriptLog -Color Red -Msg "Error: $($_.Exception.Message)"
}

# Itinerate across all SitesList
$ItemCounter = 0
ForEach($Site in $SitesList){
    # Adding notification and logs
    $PercentComplete = [math]::Round($ItemCounter/$SitesList.Count * 100, 2)
    Add-ScriptLog -Color Yellow -Msg "$($PercentComplete)% Completed - Checking Site: $($Site.Title)"
    $ItemCounter++

    Try{
        Set-PnPTenantSite -Url $Site.Url -Owners $SiteCollAdmin -ErrorAction Stop
    }
    Catch{
        Add-ScriptLog -Color Red -Msg "Error adding user as Site Collection Administrator for Site Collection '$($Site.Title)'"
        Add-ReportRecord -SiteURL $Site.Url -Remarks $_.Exception.Message
    }

    Find-IRMLibray -SiteURL $Site.Url

    $SubSites = Get-PnPSubWeb -Recurse

    ForEach($Site in $SubSites) {
        Find-IRMLibray -SiteURL $Site.Url
    }

    Remove-PnPSiteCollectionAdmin -Owners $SiteCollAdmin
}
# Close status notification
$PercentComplete = [math]::Round($ItemCounter/$SitesList.Count * 100, 1)
Add-ScriptLog -Color Cyan -Msg "$($PercentComplete)% Completed - Finished running script"
Add-ScriptLog -Color Cyan -Msg "Report generated at at $($ReportOutput)"
```

<br>

## Preservation Hold Library across all sites
```powershell
################################################################
# PARAMETERS TO BE CHANGED TO MATCH CURRENT CASE
################################################################
$AdminSiteURL = "https://Domain-admin.sharepoint.com/"
$SiteCollAdmin = "admin@email.com"



################################################################
# REPORT AND LOGS FUNCTIONS
################################################################

function Add-ReportRecord {
    param (
        $SiteUrl,
        $List,
        $Size,
        $Remarks
    )

    $Record = New-Object PSObject -Property ([ordered]@{
        SiteURL = $SiteURL
        Title = $List.Title
        ListType = $List.BaseType
        ListDefaultViewUrl = $List.DefaultViewUrl
        SizeGb = [Math]::Round(($Size/1GB), 2)
        Remarks = $Remarks
        })
    
    $Record | Export-Csv -Path $ReportOutput -NoTypeInformation -Append
}

Function Add-ScriptLog {
    param (
        $Color,
        $Msg,
        $Size,
        $Remarks
    )
    $Date = Get-Date -Format "yyyy/MM/dd HH:mm"
    $Msg = $Date + " - " + $Msg
    Add-Content -Path $LogsOutput -Value $Msg
    Write-host -f $Color $Msg
}

$Date = Get-Date -Format "yyyyMMdd_HHmmss"
$ReportName = "ListReport"
$FolderName = $Date + "_" + $ReportName
$FolderPath = "$Env:USERPROFILE\Documents\"
New-Item -Path $FolderPath -Name $FolderName -ItemType "directory"
$ReportOutput = $FolderPath + $FolderName + "\" + $ReportName + ".csv"

$LogsName = $ReportName + "_Logs.txt"
$LogsOutput = $FolderPath + $FolderName + "\" + $LogsName

Add-ScriptLog -Color Cyan -Msg "Report will be generated at $($ReportOutput)"



#################################################################
# SCRIPT LOGIC
#################################################################
Function Find-SiteLists($SiteURL)
{
    Connect-PnPOnline -Url $SiteURL -Interactive

    try{
        $PHL = Get-PnPList -Identity "Preservation Hold Library"

        $Metrics = Get-PnPFolderStorageMetric -List $PHL.Title
        Add-ReportRecord -SiteUrl $SiteURL -List $PHL -Size $Metrics.TotalSize
    }
    catch{
        Add-ReportRecord -SiteUrl $SiteURL -Remarks "Site has no PHL"
    }

}


try {
    Connect-PnPOnline -Url $AdminSiteURL -Interactive -ErrorAction Stop
    Add-ScriptLog -Color Cyan -Msg "Connected to SharePoint Admin Center"

    $collSiteCollections = Get-PnPTenantSite -ErrorAction Stop | Where-Object{ $_.Title -notlike "" -and $_.Template -notlike "*Redirect*" }
    Add-ScriptLog -Color Cyan -Msg "Collected Site Collections: $($collSiteCollections.count)"
}
catch {
    Add-ScriptLog -Color Red -Msg "Error: $($_.Exception.Message)"
    break
}

$ItemCounter = 0
ForEach($oSite in $collSiteCollections) {

    $PercentComplete = [math]::Round($ItemCounter/$collSiteCollections.Count * 100, 2)
    Add-ScriptLog -Color Yellow -Msg "$($PercentComplete)% Completed - Processing Site: $($oSite.Url)"
    $ItemCounter++

    Try{
        Set-PnPTenantSite -Url $oSite.Url -Owners $SiteCollAdmin -ErrorAction Stop

        Find-SiteLists -SiteURL $oSite.Url

        $collSubSites = Get-PnPSubWeb -Recurse

        ForEach($oSubsite in $collSubSites) {
            Find-SiteLists -SiteURL $oSubsite.Url
        }
    }
    Catch{
        Add-ScriptLog -Color Red -Msg "Error while processing Item '$($oSite.Url)"
        Add-ScriptLog -Color Red -Msg "Error message: '$($_.Exception.Message)'"
        Add-ScriptLog -Color Red -Msg "Error trace: '$($_.Exception.ScriptStackTrace)'"
        Add-ReportRecord -SiteUrl $SiteURL -Remarks $_.Exception.Message
    }
}
Add-ScriptLog -Color Cyan -Msg "100% Completed - Finished running script"
Add-ScriptLog -Color Cyan -Msg "Report generated at at $($ReportOutput)"

```