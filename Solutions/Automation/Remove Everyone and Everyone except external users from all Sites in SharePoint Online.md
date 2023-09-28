#Automation #PowerShell #PnP #SiteCollection #Subsite #Everyone #EveryoneExceptExternalUsers

<br>

## Deletion and Report

```powershell
################################################################
# PARAMETERS TO BE CHANGED TO MATCH CURRENT CASE
################################################################
$AdminSiteURL = "https://<Domain>-admin.sharepoint.com" # SharePoint Admin Center URL
$SiteCollAdmin = "admin@email.com" # SharePoint Admin Account



################################################################
# REPORT AND LOGS FUNCTIONS
################################################################

function Add-ReportRecord {
    param (
        
    )

    $Record = New-Object PSObject -Property ([ordered]@{

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
$FolderPath = "$Env:USERPROFILE\Documents\SPOSolutions\"
$Date = Get-Date -Format "yyyyMMddHHmmss"
$ReportName = "RemoveEveryone"
$FolderName = $Date + "_" + $ReportName
New-Item -Path $FolderPath -Name $FolderName -ItemType "directory"

# Files
$ReportOutput = $FolderPath + $FolderName + "\" + $FolderName + "_report.csv"
$LogsOutput = $FolderPath + $FolderName + "\" + $FolderName + "_Logs.txt"

Add-ScriptLog -Color Cyan -Msg "Report will be generated at $($ReportOutput)"



#################################################################
# SCRIPT LOGIC
#################################################################

function Remove-Groups {
    
    Remove-PnPUser -Identity "c:0(.s|true"

    Remove-PnPUser -Identity "c:0-.f|rolemanager|spo-grid-all-users/6211e9b2-aab7-472b-a320-8d5bb52ec068"

}

try {
    Connect-PnPOnline -Url $AdminSiteURL -Interactive -ErrorAction Stop
    Add-ScriptLog -Color Cyan -Msg "Connected to SharePoint Admin Center"

    $collSiteCollections = Get-PnPTenantSite -ErrorAction Stop | Where-Object{ ($_.Title -notlike "" -and $_.Template -notlike "*Redirect*") }
    Add-ScriptLog -Color Cyan -Msg "Collected Site Collections: $($collSiteCollections.count)"
}
catch {
    Add-ScriptLog -Color Red -Msg "Error: $($_.Exception.Message)"
    break
}

$ItemCounter = 0
$ItemCounterStep = 1 / $collSiteCollections.Count
ForEach($oSite in $collSiteCollections) {

    $PercentComplete = [math]::Round($ItemCounter/$collSiteCollections.Count * 100, 2)
    Add-ScriptLog -Color Yellow -Msg "$($PercentComplete)% Completed - Processing Site Collection: $($oSite.URL)"
    $ItemCounter++

    Try {
        Set-PnPTenantSite -Url $oSite.Url -Owners $SiteCollAdmin -ErrorAction Stop

        Connect-PnPOnline -Url $oSite.Url -Interactive

        Remove-Groups

        $collSubsites = Get-PnPSubWeb -Recurse -ErrorAction Stop
        ForEach($oSubsite in $collSubsites) {
            
            $PercentComplete = [math]::Round( $PercentComplete + ( ($ItemCounterStep / ($collSubsites.Count + 1)) * 100 ), 2 )
            Add-ScriptLog -Color Yellow -Msg "$($PercentComplete)% Completed - Processing Subsite: $($oSubsite.Url)"

            Connect-PnPOnline -Url $oSubsite.URL -Interactive

            Remove-Groups
        }
    }
    Catch {
        Add-ScriptLog -Color Red -Msg "Error while processing Site Collection '$($oSite.Url)'"
        Add-ScriptLog -Color Red -Msg "Error message: '$($_.Exception.Message)'"
        Add-ScriptLog -Color Red -Msg "Error Script Line: '$($_.InvocationInfo.ScriptLineNumber)'"
    }

    Connect-PnPOnline -Url $oSite.Url -Interactive
    Remove-PnPSiteCollectionAdmin -Owners $SiteCollAdmin
}
$PercentComplete = [math]::Round($ItemCounter/$collSiteCollections.Count * 100, 1)
Add-ScriptLog -Color Cyan -Msg "$($PercentComplete)% Completed - Finished running script"
Add-ScriptLog -Color Cyan -Msg "Report generated at at $($ReportOutput)"
```
## Basic Deletion
```powershell
#Define Parameters
$AdminSiteURL= "https://DOMAIN-admin.sharepoint.com"
$ItemCounter = 0

#Get Credentials to connect
$Cred  = Get-Credential

#Connect to Services
Connect-PnPOnline -Url $AdminSiteURL –Credential $Cred

#Get all Sites and itinerate
$Sites = Get-PnPTenantSite | Where{ ($_.Title -notlike "" -and $_.Template -notlike "*Redirect*" -and $_.Url -notlike "*my.sharepoint.com*") }
Write-Host -f Cyan $Sites.Count
ForEach($Site in $Sites){

    #Status notification
    $ItemCounter++
    $ItemProcess = [math]::Round($ItemCounter/$Sites.Count*100,1)
    Write-Progress -PercentComplete $ItemProcess -Activity "Processing $($ItemProcess)%" -Status "Site '$($Site.Url)"

    Try{
        # REMOVE GROUP FROM SITE COLLECTION
        Connect-PnPOnline -Url $Site.url –Credential $Cred

        Remove-PnPUser -Identity "Everyone" -Force
        Remove-PnPUser -Identity "Everyone except external users" -Force

        Try{
            # REMOVE GROUP FROM SUBSITES
            $SubSites = Get-PnPSubWeb -Recurse -Includes HasUniqueRoleAssignments
            ForEach($SubSite in $SubSites){

                # Subsite with unique permissions
                if ($SubSite.HasUniqueRoleAssignments){
                    Connect-PnPOnline -Url $SubSite.url –Credential $Cred

                    Remove-PnPUser -Identity "Everyone" -Force
                    Remove-PnPUser -Identity "Everyone except external users" -Force
                }
                Write-Host -f Green $SubSite.url"COMPLETED"
            }
        }
        Catch{
            Write-Host -f Red $SubSite.url"ERROR"
        }
        
        Write-Host -f Green $Site.url"COMPLETED"
    }
    Catch{
        Write-Host -f Red $Site.url"ERROR"
    }
}
#Close status notification
Write-Progress -Activity "Processing $($ItemProcess)%" -Status "Site '$($Site.URL)"
```