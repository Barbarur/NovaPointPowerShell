#Report #SharePointOnline #Privacy #PnP #PowerShell #SharePointOnline

<br>

## Microsoft 365 Groups Privacy

```powershell
#################################################################
# DEFINE PARAMETERS FOR THE CASE
#################################################################



#################################################################
# REPORT AND LOGS FUNCTIONS
#################################################################

Function Add-ReportRecord {
    param (
        $Group
    )

    $Record = New-Object PSObject -Property ([ordered]@{
        Case = $Group.Name
        HoldPolicy = $Group.SharePointSiteUrl
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
$ReportName = "PublicGroups"
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

    Connect-ExchangeOnline -ErrorAction Stop
    Add-ScriptLog -Color Cyan -Msg "Connected to EXO"

    $collGroups = Get-UnifiedGroup | Where-Object { $_.AccessType -eq "Public" }
    Add-ScriptLog -Color Cyan -Msg "Groups ollected: $($collGroups.Count)"

}
catch {

    Add-ScriptLog -Color Red -Msg "Error message: '$($_.Exception.Message)'"
    Add-ScriptLog -Color Red -Msg "Error trace: '$($_.Exception.ScriptStackTrace)'"
    Disconnect-ExchangeOnline
    break

}

$ItemCounter = 0
foreach($oGroup in $collGroups) {

    Add-ReportRecord -Group $oGroup
    $ItemCounter++

}
if($collGroups.Count -ne 0) {

    $PercentComplete = [math]::Round( $ItemCounter/$collGroups.Count * 100, 2 )
    Add-ScriptLog -Color Cyan -Msg "$($PercentComplete)% Completed - Finished running script"

}
Add-ScriptLog -Color Cyan -Msg "Report generated at at $($ReportOutput)"
Disconnect-ExchangeOnline
```

<br>

## Sites Privacy

```powershell
#Define Parameters
$AdminSiteURL= "https://DOMAIN-admin.sharepoint.com"
$ReportOutput = "C:\SitesOwnerPrivacyReport.csv"

#Get Credentials to connect
$Cred  = Get-Credential


#Connect to SharePoint Online Admin Center
Connect-PnPOnline -Url $AdminSiteURL –Credential $Cred
Connect-AzureAD –Credential $Cred
Connect-ExchangeOnline –Credential $Cred


#Get owners of each Site
$Global:Results = @()
$ItemCounter = 0 

#Function to add Owners to the Report
Function Add-Report($OwnerType, $OwnerName, $OwnerEmail){
    $Global:Results += New-Object PSObject -Property ([ordered]@{
        SiteName               = $Site.Title
        SiteURL                = $Site.Url
        Privacy                = $Privacy
        OwnerType              = $OwnerType
        OwnerQty               = $OwnerQty
        OwnerName              = $OwnerName
        OwnerEmail             = $OwnerEmail
        StorageTotalGB         = ('{0:N2}' -f ($Site.StorageQuota/1024))
        StorageUsedGB          = ('{0:N2}' -f ([math]::Round($Site.StorageUsageCurrent/1024,2)))
        StorageFreeGB          = ('{0:N2}' -f (($Site.StorageQuota-$Site.StorageUsageCurrent)/1024))
        })
    }


#Get all Sites and itinerate
$Sites = Get-PnPTenantSite | Where{ ($_.Title -notlike "") }
ForEach($Site in $Sites)
    {
    #Status notification
    $ItemCounter++
    $ItemProcess = [math]::Round($ItemCounter/$Sites.Count*100,1)
    Write-Progress -PercentComplete ($ItemCounter / ($Sites.Count) * 100) -Activity "Processing $($ItemProcess)%" -Status "Site '$($Site.URL)"
    $Msg = "Collecting Owners for Site '{0}'... " -f $Site.Title
    Write-Host -f Yellow $Msg -NoNewline

    $OwnerQty = "Single Owner"

    #If Owner is a MS365 GROUP
    If($Site.GroupID -notLike "00000000-0000-0000-0000-000000000000")
        {
        Try
            {
            $Owners = Get-AzureADGroupOwner -ObjectId $Site.GroupId

            $Privacy = Get-UnifiedGroup -Identity $Site.Title

            $Privacy = $Privacy.AccessType

            If($Owners.Count -ne 1) {$OwnerQty = "Multiple Owners"}

            ForEach($Owner in $Owners.UserPrincipalName)
                {
                $User = Get-AzureADUser -ObjectId $Owner
                Add-Report -OwnerType "MS365 Group" -OwnerName $User.DisplayName -OwnerEmail $User.Mail 
                }
            }
        Catch
            {
            Add-Report -OwnerType "MS365 Group" -OwnerName "DELETED GROUP" -OwnerEmail ""
            }
        }
    #If Owner is a USER
    Else
        {
        $Privacy = "Private"
        If($Site.Owner.Length -eq 0)
            {
            Add-Report -OwnerType "User" -OwnerName "NO OWNER" -OwnerEmail ""
            }
        Else
            {
            Try
                {
                $User = Get-AzureADUser -ObjectId $Site.Owner
                Add-Report -OwnerType "User" -OwnerName $User.DisplayName -OwnerEmail $User.Mail
                }
            Catch
                {
                Add-Report -OwnerType "User" -OwnerName "DELETED USER" -OwnerEmail ""
                }
            }
        }
    #Status notification
    Write-Host -f Green "COMPLETED!"
    }

#Close status notification
Write-Progress -Activity "Processing $($ItemProcess)%" -Status "Site '$($Site.URL)"

#Export the results to CSV
If (Test-Path $ReportOutput) { Remove-Item $ReportOutput }
$Global:Results | Export-Csv -Path $ReportOutput -NoTypeInformation
Write-host -b Green "Report Generated Successfully!"
Write-host -f Green $ReportOutput

Disconnect-ExchangeOnline
```