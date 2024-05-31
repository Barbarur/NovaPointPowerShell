#Automation #Versioning #PnP #PowerShell #SharePointOnline

<br>

```powershell
################################################################
# PARAMETERS TO BE CHANGED TO MATCH CURRENT CASE
################################################################
$AdminSiteURL="https://<DOMAIN>-admin.sharepoint.com"
$SiteCollAdmin = "<EMAIL@EMAIL.COM>"
$MajorLimitLibrary = 100
$MajorLimitList = 100



################################################################
# REPORT AND LOGS FUNCTIONS
################################################################
# Add new record on the report
Function Add-ReportRecord($SiteURL, $ListName, $Status, $Remarks) {
    $Record = New-Object PSObject -Property ([ordered]@{
        "SiteURL" = $SiteURL
        "ListName" = $ListName
        "SetVersionStatus" = $Status
        "Remarks" = $Remarks
        })
    
    $Record | Export-Csv -Path $ReportOutput -NoTypeInformation -Append
}

# Add Log of the Script
Function Add-ScriptLog($Color, $Msg) {
    $Date = Get-Date -Format "yyyy/MM/dd HH:mm"
    $Msg = $Date + " - " + $Msg
    Add-Content -Path $LogsOutput -Value $Msg
    Write-host -f $Color $Msg
}

# Create Report location
$FolderPath = "$Env:USERPROFILE\Documents\"
$Date = Get-Date -Format "yyyyMMddHHmmss"
$ReportName = "SetVersionLimit"
$FolderName = $Date + "_" + $ReportName
New-Item -Path $FolderPath -Name $FolderName -ItemType "directory"

# Files
$ReportOutput = $FolderPath + $FolderName + "\" + $FolderName + "_report.csv"
$LogsOutput = $FolderPath + $FolderName + "\" + $FolderName + "_Logs.txt"

Add-ScriptLog -Color Cyan -Msg "Report will be generated at $($ReportOutput)"



#################################################################
# SCRIPT LOGIC
#################################################################

function Set-VersionToAllLists( $SiteURL ) {
    
    Connect-PnPOnline -Url $SiteURL -Interactive

    $ExcludedLists = @("appdata", "appfiles", "Composed Looks", "Converted Forms", "Form Templates", "List Template Gallery", "Master Page Gallery", "Preservation Hold Library", "Project Policy Item List", "Site Assets", "Site Pages", "Solution Gallery", "Style Library", "TaxonomyHiddenList", "Theme Gallery", "User Information List", "Web Part Gallery")
    $Lists = Get-PnPList | Where-Object { ($_.Hidden -eq $False) -and ($_.Title -notin $ExcludedLists) }

    ForEach ( $List in $Lists ) {
        Write-Host -f Cyan "Modifying versioning for $($List.BaseType) '$($List.Title)'"
        Write-Host -f DarkCyan $List.DefaultViewUrl
        
        Try{
            If( $List.BaseType -eq 'DocumentLibrary' ){ Set-PnPList -Identity $List.Title -EnableVersioning $true -MajorVersions $MajorLimitLibrary -EnableMinorVersions $false }
            If( $List.BaseType -eq 'GenericList' ){ Set-PnPList -Identity $List.Title -EnableVersioning $true -MajorVersions $MajorLimitList }
            Add-ReportRecord -SiteURL $SiteURL -ListName $List.Title -Status "Success" -Remarks $_.Exception.Message
        }
        catch{
            Add-ReportRecord -SiteURL $Site.Url -ListName "Error" -Status -Remarks $_.Exception.Message
        }
    }
}

# Connect to SharePoint Site and collect subsites
try {
    Connect-PnPOnline -Url $AdminSiteURL -Interactive -ErrorAction Stop
    Add-ScriptLog -Color Cyan -Msg "Connected to SharePoint Admin Center"

    $collSiteCollections = Get-PnPTenantSite -ErrorAction Stop | Where-Object{ ($_.Title -notlike "" -and $_.Template -notlike "*Redirect*") }
    Add-ScriptLog -Color Cyan -Msg "Collected Site Collections"
    Add-ScriptLog -Color Cyan -Msg "Number of SiteCollections: $($collSiteCollections.count)"
}
catch {
    Add-ScriptLog -Color Red -Msg "Error: $($_.Exception.Message)"
    break
}

$ItemCounter = 0
$ItemCounterStep = 1 / $collSiteCollections.Count
ForEach($oSite in $collSiteCollections) {
    
    $PercentComplete = [math]::Round($ItemCounter/$collSiteCollections.Count * 100, 2)
    Add-ScriptLog -Color Yellow -Msg "$($PercentComplete)% Completed - Processing Site Collection: $($oSite.Title)"
    $ItemCounter++

    try {

        Set-PnPTenantSite -Url $oSite.Url -Owners $SiteCollAdmin -ErrorAction Stop

        Set-VersionToAllLists -SiteURL $oSite.URL

        $collSubsites = Get-PnPSubWeb -Recurse
        foreach ($oSubsite in $collSubsites) {

            $PercentComplete = [math]::Round( $PercentComplete + ( ($ItemCounterStep / $collSubsites.Count) * 100 ), 2 )
            Add-ScriptLog -Color Yellow -Msg "$($PercentComplete)% Completed - Processing Subsite: $($oSubsite.Title)"

            try {
                Set-VersionToAllLists -SiteURL $Site.url
            }
            catch {
                Add-ScriptLog -Color Red -Msg "Error while processing Subsite '$($oSubsite.Url)'"
                Add-ScriptLog -Color Red -Msg "Error message: '$($_.Exception.Message)'"
                Add-ScriptLog -Color Red -Msg "Error trace: '$($_.Exception.ScriptStackTrace)'"
                Add-ReportRecord -SiteURL $oSubsite.Url -Remarks $_.Exception.Message
            }
        }
    }
    catch {
        Add-ScriptLog -Color Red -Msg "Error while processing Site Collection '$($Site.Url)'"
        Add-ScriptLog -Color Red -Msg "Error message: '$($_.Exception.Message)'"
        Add-ScriptLog -Color Red -Msg "Error trace: '$($_.Exception.ScriptStackTrace)'"
        Add-ReportRecord -SiteURL $oSite.Url -Remarks $_.Exception.Message
    }
}

if($collSiteCollections.Count -ne 0) { 
    $PercentComplete = [math]::Round($ItemCounter/$SitesList.Count * 100, 1) 
    Add-ScriptLog -Color Cyan -Msg "$($PercentComplete)% Completed - Finished running script"
}
Add-ScriptLog -Color Cyan -Msg "Report generated at at $($ReportOutput)"
```