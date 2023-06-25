#Report

<br>

```powershell
#################################################################
# DEFINE PARAMETERS FOR THE CASE
#################################################################
$SPOAdminURL = "https://<DOMAIN>-admin.sharepoint.com/"
$GlobalAdminUPN = "<ADMIN@EMAIL.com>" # Global Admin used to run the script.


#################################################################
# REPORT AND LOGS FUNCTIONS
#################################################################

Function Add-ReportRecord {
    param (
        $Case,
        $Hold,
        $SiteTitle,
        $SiteUrl
    )

    $Record = New-Object PSObject -Property ([ordered]@{
        "Case" = $Case
        "HoldPolicy" = $Hold
        "SiteTitle" = $SiteTitle
        "SiteURL" = $SiteUrl
        })
    
    $Record | Export-Csv -Path $ReportOutput -NoTypeInformation -Append
}

Function Add-ScriptLog($Color, $Msg) {
    Write-host -f $Color $Msg
    $Date = Get-Date -Format "yyyy/MM/dd HH:mm"
    $Msg = $Date + " - " + $Msg
    Add-Content -Path $LogsOutput -Value $Msg
}

# Create Report location
$FolderPath = "$Env:USERPROFILE\Documents\SPOSolutions\"
$Date = Get-Date -Format "yyyyMMddHHmmss"
$ReportName = "HoldSitesReport"
$FolderName = $Date + "_" + $ReportName
New-Item -Path $FolderPath -Name $FolderName -ItemType "directory"

# Files
$ReportOutput = $FolderPath + $FolderName + "\" + $FolderName + "_report.csv"
$LogsOutput = $FolderPath + $FolderName + "\" + $FolderName + "_Logs.txt"

Add-ScriptLog -Color Cyan -Msg "Report will be generated at $($ReportOutput)"



#################################################################
# SCRIPT LOGIC
#################################################################

function Get-PolicySites {
    param (
        $CaseName,
        $HoldName
    )

    Add-ScriptLog -Color yellow -Msg "Processing Policy: $($HoldName)"

    $details = Get-CaseHoldPolicy -Identity $HoldName
    Add-ScriptLog -Color yellow -Msg "Sites on Hold: $($details.SharePointLocation.Count)"

    if ( $details.SharePointLocation.count -lt 1 ) {

        Add-ReportRecord -Case $CaseName -Hold $HoldName -SiteTitle "No SPO Sites on Hold" -SiteUrl "No SPO Sites on Hold"

    }
    else {

        foreach ( $location in $details.SharePointLocation ) {

            foreach ($oSite in $collSiteCollections ) {
                
                if ( $oSite.Title -eq $location) {
                
                    Add-ReportRecord -Case $CaseName -Hold $HoldName -SiteTitle $oSite.Title -SiteUrl $oSite.Url
                
                }
            }
        }
    }
}


try {
    
    Connect-IPPSSession -UserPrincipalName $GlobalAdminUPN
    Add-ScriptLog -Color Cyan -Msg "Connected to Compliance Center"

    $collCases = Get-ComplianceCase
    Add-ScriptLog -Color Cyan -Msg "Cases collected"


    Connect-PnPOnline -Url $SPOAdminURL -Interactive -ErrorAction Stop
    Add-ScriptLog -Color Cyan -Msg "Connected to SharePoint Admin Center"

    $collSiteCollections = Get-PnPTenantSite -IncludeOneDriveSites -ErrorAction Stop
    Add-ScriptLog -Color Cyan -Msg "Collected Site Collections"
    Add-ScriptLog -Color Cyan -Msg "Number of SiteCollections: $($collSiteCollections.count)"

}
catch {

    Add-ScriptLog -Color Red -Msg "Error message: '$($_.Exception.Message)'"
    Add-ScriptLog -Color Red -Msg "Error trace: '$($_.Exception.ScriptStackTrace)'"
    Disconnect-ExchangeOnline
    break

}

$ItemCounter = 0
foreach($oCase in $collCases) {

    $PercentComplete = [math]::Round($ItemCounter/$collCases.Count * 100, 2)
    Add-ScriptLog -Color Yellow -Msg "$($PercentComplete)% Completed - Processing case: $($oCase.Name)"
    $ItemCounter++

    try{

        $collHold = Get-CaseHoldPolicy -Case $oCase.Name -ErrorAction Stop
        
        if($collHold.Count -gt 0) {

            foreach($oHold in $collHold) {
                
                Get-PolicySites -CaseName $oCase.Name -HoldName $oHold.Name

            }
        }
        elseif ($null -ne $collHold -and $collHold.count -lt 0) {
            
            Get-PolicySites -CaseName $oCase.Name -HoldName $collHold.Name

        }
        else {

            Add-ScriptLog -Color Yellow -Msg "Case '$($oCase.Name)' has no Hold Policies"
            Add-ReportRecord -Case $oCase.Name -Hold "No Hold Policies" -SiteTitle "No Hold Policies" -SiteUrl "No Hold Policies"

        }
    }
    catch{
        
        Add-ScriptLog -Color Red -Msg "Error Processing case '$($oCase.Name)'"
        Add-ScriptLog -Color Red -Msg "Error message: '$($_.Exception.Message)'"
        Add-ScriptLog -Color Red -Msg "Error message: '$($_.Exception.ScriptStackTrace)'"

    }
}
if($collCases.Count -ne 0) {

    $PercentComplete = [math]::Round( $ItemCounter/$collCases.Count * 100, 2 )
    Add-ScriptLog -Color Cyan -Msg "$($PercentComplete)% Completed - Finished running script"

}
Add-ScriptLog -Color Cyan -Msg "Report generated at at $($ReportOutput)"
Disconnect-ExchangeOnline
```