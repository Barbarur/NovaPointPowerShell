#Automation #OneDrive #SharePointOnline #PnP #PowerShell #SharedLink

<br>

```powershell
################################################################
# PARAMETERS TO BE CHANGED TO MATCH CURRENT CASE
################################################################
$SiteURL= "https://DOMAIN.sharepoint.com/sites/SITENAME"



################################################################
# REPORT AND LOGS FUNCTIONS
################################################################

Function Add-ReportRecord($SiteURL, $ListName, $Item, $Link, $Action) {
    $Record = New-Object PSObject -Property ([ordered]@{
        "SiteURL" = $SiteURL
        "ListName" = $ListName
        "ItemName" = $Item["FileLeafRef"]
        "ItemPath" = $Item["FileRef"]
        "Link" = $Link
        "Action" = $Action
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
$ReportName = "RemoveSharedLinks"
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
    Connect-PnPOnline -Url $SiteURL -Interactive -ErrorAction Stop
    Add-ScriptLog -Color Cyan -Msg "Connected to SharePoint Admin Center"

    $ExcludedLists = @("appdata", "appfiles", "Composed Looks", "Converted Forms", "Form Templates", "List Template Gallery", "Master Page Gallery", "Preservation Hold Library", "Project Policy Item List", "Site Assets", "Site Pages", "Solution Gallery", "Style Library", "TaxonomyHiddenList", "Theme Gallery", "User Information List", "Web Part Gallery")
    $Lists = Get-PnPList | Where-Object {$_.Hidden -eq $False -and $_.Title -notin $ExcludedLists}
    Add-ScriptLog -Color Cyan -Msg "Collected Lists"
    Add-ScriptLog -Color Cyan -Msg "Number of Lists: $($Lists.count)"
}
catch {
    Add-ScriptLog -Color Red -Msg "Error: $($_.Exception.Message)"
    break
}

$ListCounter = 0
ForEach ($List in $Lists) {

    $PercentComplete = [math]::Round($ItemCounter/$Lists.Count * 100, 2)
    Add-ScriptLog -Color Yellow -Msg "$($PercentComplete)% Completed - Processing $($List.BaseType) '$($List.Title)'"
    $ListCounter++

    $Items = Get-PnPListItem -List $List.Title -PageSize 2000
    ForEach($Item in $Items) {

        $HasUniquePermissions = Get-PnPProperty -ClientObject $Item -Property "HasUniqueRoleAssignments"
        If($HasUniquePermissions) {

            $RoleAssignments = Get-PnPProperty -ClientObject $Item -Property RoleAssignments
            ForEach($RoleAssignment in $RoleAssignments) {

                Get-PnPProperty -ClientObject $RoleAssignment -Property RoleDefinitionBindings, Member
                
                If($RoleAssignment.Member.Title -notlike "SharingLinks*") {Continue}
                Add-ScriptLog -Color Yellow -Msg "Found SharedLink with ID: $($RoleAssignment.Member.Id). $($RoleAssignment.Member.Description)"

                try{
                    Remove-PnPGroup -Identity $RoleAssignment.Member.Title -Force
                    Add-ReportRecord -SiteURL $SiteURL -ListName $List.Title -Item $Item -Link $RoleAssignment.Member.Title -Action "Removed"
                }
                catch{
                    Add-ReportRecord -SiteURL $SiteURL -ListName $List.Title -Item $Item -Link $RoleAssignment.Member.Title -Action "Error"
                }
            }
        }
        Write-Progress -Activity "Processing item $($Item["FileLeafRef"])" -Status $ListMsg -Completed
    }
}

if($Lists.Count -ne 0) { 
    $PercentComplete = [math]::Round($ItemCounter/$Lists.Count * 100, 2) 
    Add-ScriptLog -Color Cyan -Msg "$($PercentComplete)% Completed - Finished running script"
}
Add-ScriptLog -Color Cyan -Msg "Report generated at at $($ReportOutput)"


```