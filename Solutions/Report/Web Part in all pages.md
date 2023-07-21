#Report #SharePointOnline #PowerShell #PnP #DocumentLibrary #Web-Part #Page

<br>

## PnP Modern Search

```powershell
#################################################################
# DEFINE PARAMETERS FOR THE CASE
#################################################################
$AdminSiteURL = "https://<DOMAIN>-admin.sharepoint.com" # SharePoint Admin Center Url
$SiteCollAdmin = "<ADMIN@EMAIL.com>" # Global or SharePoint Admin used for loging running the script.



#################################################################
# REPORT AND LOGS FUNCTIONS
#################################################################

function Add-ReportRecord {
    param (
        $PageRelativeUrl
    )

    $Record = New-Object PSObject -Property ([ordered]@{
        RelativeUrl = $PageRelativeUrl
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
$ReportName = "PnPPages"
$FolderName = $Date + "_" + $ReportName
New-Item -Path $FolderPath -Name $FolderName -ItemType "directory"

# Files
$ReportOutput = $FolderPath + $FolderName + "\" + $FolderName + "_report.csv"
$LogsOutput = $FolderPath + $FolderName + "\" + $FolderName + "_Logs.txt"

Add-ScriptLog -Color Cyan -Msg "Report will be generated at $($ReportOutput)"



#################################################################
# SCRIPT LOGIC
#################################################################

function Find-PnPSearchInPages {
    param (
        $SiteURL
    )

    Connect-PnPOnline -Url $SiteURL -Interactive

    Add-ScriptLog -Color DarkYellow -Msg "Start checking pages"
    try {
        $collPages = Get-PnPListItem -List "Site Pages"

        foreach($oPage in $collPages) {
            
            if($oPage.FieldValues.CanvasContent1 -match "pnp-modern-search") {
                
                Add-ReportRecord -PageRelativeUrl $oPage.FieldValues.FileRef
            }
        }
    }
    catch {
    }
    Add-ScriptLog -Color DarkYellow -Msg "Finish checking pages"
}


try {
    Connect-PnPOnline -Url $AdminSiteURL -Interactive -ErrorAction Stop
    Add-ScriptLog -Color Cyan -Msg "Connected to SharePoint Admin Center"

    #$collSiteCollections = Get-PnPTenantSite | Where-Object { ($_.Title -notlike "" -and $_.Template -notlike "*Redirect*" -and $_.Url -like "https://m365x88421522.sharepoint.com/sites/NorAqilaClassic") }
    $collSiteCollections = Get-PnPTenantSite | Where-Object { ($_.Title -notlike "" -and $_.Template -notlike "*Redirect*") }
    Add-ScriptLog -Color Cyan -Msg "Collected all Site Collections: $($collSiteCollections.Count)"
}
catch {
    Add-ScriptLog -Color Red -Msg "Error: $($_.Exception.Message)"
    break
}


$ItemCounter = 0
$ItemCounterStep = 1 / $collSiteCollections.Count
ForEach($oSite in $collSiteCollections) {

    # Adding notification and logs
    $PercentComplete = [math]::Round($ItemCounter/$collSiteCollections.Count * 100, 2)
    Add-ScriptLog -Color Yellow -Msg "$($PercentComplete)% Completed - Processing Site Collection: $($oSite.Title)"
    $ItemCounter++

    
    Try {
        Set-PnPTenantSite -Url $oSite.Url -Owners $SiteCollAdmin
        Add-ScriptLog -Color DarkYellow -Msg "$($SiteCollAdmin) added as Site Collection Admin"

        Find-PnPSearchInPages -SiteURL $oSite.Url
        
        $collSubsites = Get-PnPSubWeb -Recurse
        Add-ScriptLog -Color DarkYellow -Msg "Collected Subsites: $($collSubsites.Count)"
        
        ForEach($oSubsite in $collSubsites) {

            $PercentComplete = [math]::Round( $PercentComplete + ( ($ItemCounterStep / $collSubsites.Count) * 100 ), 2 )
            Add-ScriptLog -Color Yellow -Msg "$($PercentComplete)% Completed - Processing Subsite: $($oSubsite.Title)"

            try {
                Find-PnPSearchInPages -SiteURL $oSubsite.Url
            }
            catch {
                Add-ScriptLog -Color Red -Msg "Error while processing Subsite '$($oSubsite.Url)'"
                Add-ScriptLog -Color Red -Msg "Error message: '$($_.Exception.Message)'"
                Add-ScriptLog -Color Red -Msg "Error trace: '$($_.Exception.ScriptStackTrace)'"
            }
        }
    }
    Catch {
        Add-ScriptLog -Color Red -Msg "Error while processing Site Collection '$($oSite.Url)'"
        Add-ScriptLog -Color Red -Msg "Error message: '$($_.Exception.Message)'"
        Add-ScriptLog -Color Red -Msg "Error trace: '$($_.Exception.ScriptStackTrace)'"
    }

    Connect-PnPOnline -Url $oSite.Url -Interactive
    Remove-PnPSiteCollectionAdmin -Owners $SiteCollAdmin
    Add-ScriptLog -Color DarkYellow -Msg "$($SiteCollAdmin) removed as Site Collection Admin"
}

if($collSiteCollections.Count -ne 0) { 
    $PercentComplete = [math]::Round($ItemCounter/$collSiteCollections.Count * 100, 1) 
    Add-ScriptLog -Color Cyan -Msg "$($PercentComplete)% Completed - Finished running script"
}
Add-ScriptLog -Color Cyan -Msg "Report generated at at $($ReportOutput)"
```