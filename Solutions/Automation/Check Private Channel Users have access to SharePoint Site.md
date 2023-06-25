#Report

<br>

```powershell
#################################################################
# DEFINE PARAMETERS FOR THE CASE
#################################################################
$SiteURL = "https://<Domain>.sharepoint.com/sites/<SiteName-PrivateChannel" # SharePoint Admin Center Url
$TeamsGroupID = "GroupID"
$SharedChannelName = "<Private/SharedChannelName"



#################################################################
# REPORT AND LOGS FUNCTIONS
#################################################################

Function Add-ReportRecord {
    param (
        $User,
        $Status,
        $Remarks
        )

    $Record += New-Object PSObject -Property ([ordered]@{
        User = $User
        Status = $Status
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
$FolderPath = "$Env:USERPROFILE\Documents\SPOSolutions\Report\"
$Date = Get-Date -Format "yyyyMMddHHmmss"
$ReportName = "SiteOwnersReport"
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
    Add-ScriptLog -Color Cyan -Msg "Connected to SharePoint Site"
    $CollSiteMembers = Get-PnPUser
    Add-ScriptLog -Color Cyan -Msg "Collected users from the Site: $($SiteMembers.Count)"

    Connect-MicrosoftTeams
    Add-ScriptLog -Color Cyan -Msg "Connected to Teams"
    $CollSharedChannelMembers = Get-TeamChannelUser -GroupId $TeamsGroupID -DisplayName $SharedChannelName -Role Member
    Add-ScriptLog -Color Cyan -Msg "Collected users from the Shared Channel: $($CollSharedChannelMembers.Count)"

}
catch {

    Add-ScriptLog -Color Red -Msg "Error: $($_.Exception.Message)"
    break

}


$ItemCounter = 0
ForEach($Member in $CollSharedChannelMembers) {

    # Adding notification and logs
    $PercentComplete = [math]::Round($ItemCounter/$CollSharedChannelMembers.Count * 100, 2)
    Add-ScriptLog -Color Yellow -Msg "$($PercentComplete)% Completed - Processing Shared Channel Member: $($Member.User)"
    $ItemCounter++
    
    $IsMatch = $false

    Try {

        foreach($SiteMember in $CollSiteMembers)
        {
            if ( "$($SiteMember.LoginName)" -like "*$($Member.User)*" ) {
                
                $IsMatch = $true

            }
        }

        if ($IsMatch) {
            Add-ReportRecord -User $Member.User -Status "User Found on Site"
        }
        else {
            Add-ReportRecord -User $Member.User -Status "NOT FOUND"
        }

    }
    Catch {
        Add-ScriptLog -Color Red -Msg "Error while processing Shared Channel Member: $($Member.User)"
        Add-ScriptLog -Color Red -Msg "Error message: '$($_.Exception.Message)'"
        Add-ScriptLog -Color Red -Msg "Error trace: '$($_.Exception.ScriptStackTrace)'"
        Add-ReportRecord -User $Member.User -Remarks $_.Exception.Message
    }
}

if($CollSharedChannelMembers.Count -ne 0) { 
    $PercentComplete = [math]::Round($ItemCounter/$CollSharedChannelMembers.Count * 100, 1) 
    Add-ScriptLog -Color Cyan -Msg "$($PercentComplete)% Completed - Finished running script"
}
Add-ScriptLog -Color Cyan -Msg "Report generated at at $($ReportOutput)"
```