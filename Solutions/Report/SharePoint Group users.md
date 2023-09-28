#Report #SharePointOnline #OneDrive #PowerShell #PnP #AssociatedGroup  #Users

<br>

## Users in the Associated SharePoint Groups

```powershell
################################################################
# PARAMETERS TO BE CHANGED TO MATCH CURRENT CASE
################################################################

# SharePoint Admin Center URL
$AdminSiteURL = "https://<Domain>-admin.sharepoint.com"

# SharePoint Admin Account
$SiteCollAdmin = "<admin@email.com>" 

# Path to the CSV file with the list of Site URLs. File should have a single column with header "Url".
# Leave it empty to go through all Site Collections, this will make the script to run long time and high provability tp crash
$CSVFile = "" 



################################################################
# REPORT AND LOGS FUNCTIONS
################################################################

function Add-ReportRecord {
    param (
        $SiteUrl,
        $Owners,
        $Members,
        $Visitors,
        $Remarks
    )

    $Record = New-Object PSObject -Property ([ordered]@{
        SiteUrl = $SiteUrl
        Owners = $Owners
        Members = $Members
        Visitors = $Visitors
        Remarks = $Remarks
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
$ReportName = "SharePointGroupMembersReport"
$FolderName = $Date + "_" + $ReportName
New-Item -Path $FolderPath -Name $FolderName -ItemType "directory"

# Files
$ReportOutput = $FolderPath + $FolderName + "\" + $FolderName + "_report.csv"
$LogsOutput = $FolderPath + $FolderName + "\" + $FolderName + "_Logs.txt"

Add-ScriptLog -Color Cyan -Msg "Report will be generated at $($ReportOutput)"



#################################################################
# SCRIPT LOGIC
#################################################################

function Find-SiteMembership {
    param (
        $SiteUrl
    )
    
    Add-ScriptLog -Color White -Msg "Finding SharePoint Groups in Site '$($Site.Url)'"

    $AssociatedOwnerGroup = Get-PnPGroup -AssociatedOwnerGroup
    $Owners = Find-SharePointGroupMembers -SiteUrl $SiteUrl -SharePointGroupTitle $AssociatedOwnerGroup.Title

    $AssociatedMemberGroup = Get-PnPGroup -AssociatedMemberGroup
    $Members = Find-SharePointGroupMembers -SiteUrl $SiteUrl -SharePointGroupTitle $AssociatedMemberGroup.Title

    $AssociatedVisitorGroup = Get-PnPGroup -AssociatedVisitorGroup
    $Visitors = Find-SharePointGroupMembers -SiteUrl $SiteUrl -SharePointGroupTitle $AssociatedVisitorGroup.Title

    Add-ReportRecord -SiteUrl $SiteUrl -Owners $Owners -Members $Members -Visitors $Visitors
}


function Find-SharePointGroupMembers {
    param (
    $SiteUrl,    
    $SharePointGroupTitle
    )

    $collSharePointGroupUsers = ''
    
    Add-ScriptLog -Color White -Msg "Finding users in SharePoint group '$($SharePointGroupTitle)'"

    try { $collSharePointGroupMembers = Get-PnPGroupMember -Identity $SharePointGroupTitle }
    catch {
        Add-ScriptLog -Color Red -Msg "Error while finding users in SharePoint group '$($SharePointGroupTitle)'"
        Add-ScriptLog -Color Red -Msg "Error message: '$($_.Exception.Message)'"
        Add-ScriptLog -Color Red -Msg "Error Script Line: '$($_.InvocationInfo.ScriptLineNumber)'"
        Add-ReportRecord -SiteUrl $SiteUrl -Remarks $_.Exception.Message
        return ''
    }

    foreach ($oSharePointGroupMember in $collSharePointGroupMembers) {

        Add-ScriptLog -Color White -Msg "Processing member '$($oSharePointGroupMember.Title)' $($oSharePointGroupMember.PrincipalType) $($oSharePointGroupMember.UserPrincipalName)"

        if ($oSharePointGroupMember.PrincipalType -eq "SecurityGroup") {

            $collSharePointGroupUsers += Find-SecurityGroupMembers -SiteUrl $SiteUrl -GroupName $oSharePointGroupMember.Title -GroupID $oSharePointGroupMember.LoginName
        }
        elseif ($oSharePointGroupMember.PrincipalType -eq "User" -and $oSharePointGroupMember.UserPrincipalName) {
            
            $collSharePointGroupUsers += "$($oSharePointGroupMember.UserPrincipalName); "
        }
    }

    return $collSharePointGroupUsers
}


function Find-SecurityGroupMembers {
    param (
        $SiteUrl,
        $GroupName,
        $GroupID
    )

    $collSecurityGroupUsers = ''

    $ExcludedGroups = @("Global Administrator", "SharePoint Administrator", "Everyone", "Everyone except external users", "System Account" )

    If( $GroupName -in $ExcludedGroups ) { Continue }

    Add-ScriptLog -Color White -Msg "Finding users in Security Group '$($GroupName)' '$($GroupID)'"
    
    $GroupIDClean = Format-LoginName -LoginName $GroupID
    
    if([string]::IsNullOrWhiteSpace($GroupIDClean)) { return }

    try {
        if($GroupID -clike '*_o'){
    
            Add-ScriptLog -Color White -Msg "Getting AAD Group Owners '$($GroupName)' '$($GroupIDClean)'"
            $GroupUsers = Get-PnPAzureADGroupOwner -Identity $GroupIDClean
        }
        else{
    
            Add-ScriptLog -Color White -Msg "Getting AAD Group Members '$($GroupName)' '$($GroupIDClean)'"
            $GroupUsers = Get-PnPAzureADGroupMember -Identity $GroupIDClean
        }
    }
    catch {
        Add-ScriptLog -Color Red -Msg "Error while finding users in Security Group '$($GroupName)' '$($GroupID)'"
        Add-ScriptLog -Color Red -Msg "Error message: '$($_.Exception.Message)'"
        Add-ScriptLog -Color Red -Msg "Error Script Line: '$($_.InvocationInfo.ScriptLineNumber)'"
        Add-ReportRecord -SiteUrl $SiteUrl -Remarks $_.Exception.Message
        return ''
    }

    foreach ($oUser in $GroupUsers) {
        
        Add-ScriptLog -Color White -Msg "Processing member '$($oUser.UserPrincipalName)'"

        if ($oUser.Type -eq "User" -and $oUser.UserPrincipalName) {

            $collSecurityGroupUsers += "$($oUser.UserPrincipalName); "
        }
        elseif ($oUser.Type -eq "Group") {

            $collSecurityGroupUsers += Find-SecurityGroupMembers -SiteUrl $SiteUrl -GroupName $oUser.DisplayName -GroupID $oUser.UserPrincipalName
        }
    }

    return $collSecurityGroupUsers
}


function Format-LoginName {
    param (
        $LoginName
    )
    $GroupID = $LoginName
    $GroupID = $GroupID -replace ('_o', '')
    $GroupID = $GroupID -replace ('c:0o.c|federateddirectoryclaimprovider|', '')
    $GroupID = $GroupID -replace ('c:0t.c|tenant|', '')
    $GroupID = $GroupID.Trim('|')

    Add-ScriptLog -Color White -Msg "Clean Security Group ID '$($GroupID)'"

    Return $GroupID
}


try {
    Connect-PnPOnline -Url $AdminSiteURL -Interactive -ErrorAction Stop
    Add-ScriptLog -Color Cyan -Msg "Connected to SharePoint Admin Center"
    
    if($CSVFile) {
        $collSiteCollections = Import-CSV $CSVFile
    }
    else {
        $collSiteCollections = Get-PnPTenantSite -ErrorAction Stop | Where-Object{ ($_.Title -notlike "" -and $_.Template -notlike "*Redirect*") }
    }
    Add-ScriptLog -Color Cyan -Msg "Collected Site Collections: $($collSiteCollections.count)"
}
catch {
    Add-ScriptLog -Color Red -Msg "Error: $($_.Exception.Message)"
    break
}


$ItemCounter = 0
ForEach($oSite in $collSiteCollections) {

    $PercentComplete = [math]::Round($ItemCounter/$collSiteCollections.Count * 100, 2)
    Add-ScriptLog -Color Yellow -Msg "$($PercentComplete)% Completed - Processing Site Collection: $($oSite.URL)"
    $ItemCounter++

    Try {
        Set-PnPTenantSite -Url $oSite.Url -Owners $SiteCollAdmin -ErrorAction Stop

        Connect-PnPOnline -Url $oSite.Url -Interactive

        Find-SiteMembership -SiteUrl $oSite.Url
    }
    Catch {
        Add-ScriptLog -Color Red -Msg "Error while processing Site Collection '$($oSite.Url)'"
        Add-ScriptLog -Color Red -Msg "Error message: '$($_.Exception.Message)'"
        Add-ScriptLog -Color Red -Msg "Error Script Line: '$($_.InvocationInfo.ScriptLineNumber)'"
        Add-ReportRecord -SiteUrl $oSite.Url -Remarks $_.Exception.Message
    }

    Connect-PnPOnline -Url $oSite.Url -Interactive
    Remove-PnPSiteCollectionAdmin -Owners $SiteCollAdmin
}

$PercentComplete = [math]::Round($ItemCounter/$collSiteCollections.Count * 100, 1)
Add-ScriptLog -Color Cyan -Msg "$($PercentComplete)% Completed - Finished running script"
Add-ScriptLog -Color Cyan -Msg "Report generated at at $($ReportOutput)"
```
