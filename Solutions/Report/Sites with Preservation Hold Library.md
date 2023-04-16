#Report #SharePointOnline #Permissions #PnP #PowerShell #SharePointOnline  #PHL #PreservationHolLibrary 

<br>

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
Function Add-ReportRecord($SiteURL, $ListName, $ListPath, $ListSize, $Remarks)
{
    $Record = New-Object PSObject -Property ([ordered]@{
        "Site URL" = $SiteURL
        "Library Name" = $ListName
        "LibraryPath" = $ListPath -Replace ('Forms/AllItems.aspx','') -Replace ('AllItems.aspx','')
        "LibrarySizeMB" = $ListSize
        "Remarks" = $Remarks
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
$FolderPath = "$Env:USERPROFILE\Documents\NovaPoint\Report\"
$Date = Get-Date -Format "yyyyMMddHHmmss"
$ReportName = "PHLReport"
$FolderName = $Date + "_" + $ReportName
New-Item -Path $FolderPath -Name $FolderName -ItemType "directory"

# Files
$ReportOutput = $FolderPath + $FolderName + "\" + $FolderName + "_report.csv"
$LogsOutput = $FolderPath + $FolderName + "\" + $FolderName + "_Logs.txt"

Add-ScriptLog -Color Cyan -Msg "Report will be generated at $($ReportOutput)"



#################################################################
# SCRIPT LOGIC
#################################################################

Function Find-PHL($SiteURL)
{
    Connect-PnPOnline -Url $SiteURL -Interactive -ErrorAction Stop

    $ListsList = Get-PnPList | Where-Object { $_.Title -eq "Preservation Hold Library" }

    if ($null -eq $ListsList) {
        Add-ReportRecord -SiteURL $SiteURL -ListName "NA" -ListPath "NA" -ListSize "NA" -Remarks "This site doesn't have PHL"
    }
    else {
        ForEach($List in $ListsList) {
            $collItems = Get-PnPListItem -List $List.Title -PageSize 2000
            
            $ItemsTotalSize = 0
            foreach ($Item in $collItems){
                $ItemsTotalSize += $Item["SMTotalSize"].LookupId
            }
            
            $ItemsTotalSize     = [Math]::Round(($ItemsTotalSize/1MB),1)
            Add-ReportRecord -SiteURL $SiteURL -ListName $List.Title -ListPath $List.DefaultViewUrl -ListSize $ItemsTotalSize -Remarks ""
        }
    }
}

try {
    Connect-PnPOnline -Url $AdminSiteURL -Interactive -ErrorAction Stop
    Add-ScriptLog -Color Cyan -Msg "Connected to SharePoint Admin Center"
    $SitesList = Get-PnPTenantSite -ErrorAction Stop | Where-Object{ ($_.Title -notlike "" -and $_.Template -notlike "*Redirect*") }
    Add-ScriptLog -Color Cyan -Msg "Collected Site Collections"
    Add-ScriptLog -Color Cyan -Msg "Number of SiteCollections: $($SitesList.count)"
}
catch {
    Add-ScriptLog -Color Red -Msg "Error: $($_.Exception.Message)"
    break
}

$ItemCounter = 0
ForEach ( $Site in $SitesList ) {
    # Adding notification and logs
    $PercentComplete = [math]::Round($ItemCounter/$SitesList.Count * 100, 2)
    Add-ScriptLog -Color Yellow -Msg "$($PercentComplete)% Completed - Checking Site Collection: $($Site.Title)"
    $ItemCounter++

    Try{
        Set-PnPTenantSite -Url $Site.Url -Owners $SiteCollAdmin -ErrorAction Stop

        Find-PHL -SiteURL $Site.Url

        $SubSites = Get-PnPSubWeb -Recurse

        ForEach($Site in $SubSites) {
            try {
                Add-ScriptLog -Color DarkYellow -Msg "$($PercentComplete)% Completed - Checking Subsite: $($Site.Title)"
                Find-PHL -SiteURL $Site.Url
            }
            catch {
                Add-ScriptLog -Color Red -Msg "Error while processing Subsite '$($Site.Path)'"
                Add-ReportRecord -SiteURL $Site.Url -ListName "Error" -ListPath "Error" -ListSize "Error" -Remarks $_.Exception.Message
            }
        }
    }
    Catch{
        Add-ScriptLog -Color Red -Msg "Error while processing Site Collection '$($Site.Path)': $($Site.Path)"
        Add-ReportRecord -SiteURL $Site.Url -ListName "Error" -ListPath "Error" -ListSize "Error" -Remarks $_.Exception.Message
    }
}
if($collSiteCollections.Count -ne 0) { 
    $PercentComplete = [math]::Round($ItemCounter/$SitesList.Count * 100, 1) 
    Add-ScriptLog -Color Cyan -Msg "$($PercentComplete)% Completed - Finished running script"
}
Add-ScriptLog -Color Cyan -Msg "Report generated at at $($ReportOutput)"
```