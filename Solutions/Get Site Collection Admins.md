#Report #SharePointOnline #OneDrive #PowerShell #PnP #SPOService #SiteCollection #SiteAdmin 

<br>

## Using PnP: Get all Site Collection Administrators

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
        $Site,
        $AccessType,
        $UserEmail,
        $Remarks
    )

    $Record = New-Object PSObject -Property ([ordered]@{
        
        SiteName = $Site.Title
        SiteURL = $Site.url
        
        AccessType = $AccessType
        UserEmail = $UserEmail
        
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
$ReportName = "AdminsAllReport"
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

Function Get-SGUsers {
    param (
        $Site,
        $Group
    )

    $collSecurityGroupUsers = ''

    
    $ExcludedGroups = @("Global Administrator", "SharePoint Administrator", "Everyone", "Everyone except external users", "System Account" )
    If( $Group.Title -in $ExcludedGroups ) { Continue }
    
    Add-ScriptLog -Color White -Msg "Checking Security Group $($Group.Title) $($Group.LoginName)"

    Try {
        If ($Group.LoginName -clike '*_o') {
            $GroupID = Get-ObjectID -LoginName $Group.LoginName
            $GroupUsers = Get-PnPAzureADGroupOwner -Identity $GroupID
        }
        Else {
            $GroupID = Get-ObjectID -LoginName $Group.LoginName
            $GroupUsers = Get-PnPAzureADGroupOwner -Identity $GroupID
        }
    }
    Catch{
        Add-ScriptLog -Color Red -Msg "Error while finding users in Security Group '$($GroupName)' '$($GroupID)'"
        Add-ScriptLog -Color Red -Msg "Error message: '$($_.Exception.Message)'"
        Add-ScriptLog -Color Red -Msg "Error Script Line: '$($_.InvocationInfo.ScriptLineNumber)'"
        Add-ReportRecord -SiteUrl $SiteUrl -Remarks $_.Exception.Message
        return ''
    }

    foreach ($oUser in $GroupUsers) {
        
        if ($oUser.Type -eq "User" -and $oUser.UserPrincipalName) {

            $collSecurityGroupUsers += "$($oUser.UserPrincipalName); "
        }
        elseif ($oUser.Type -eq "Group") {

            $collSecurityGroupUsers += Find-SecurityGroupMembers -SiteUrl $SiteUrl -GroupName $oUser.DisplayName -GroupID $oUser.UserPrincipalName
        }
    }

    return $collSecurityGroupUsers
}

Function Get-ObjectID {
    param (
        $LoginName
    )
    
    $GroupID = $LoginName
    $GroupID = $GroupID -replace ('_o', '')
    $GroupID = $GroupID -replace ('c:0o.c|federateddirectoryclaimprovider|', '')
    $GroupID = $GroupID -replace ('c:0t.c|tenant|', '')
    $GroupID = $GroupID.Trim('|')

    Return $GroupID
}


try {
    Connect-PnPOnline -Url $AdminSiteURL -ClientId $ClientId -Interactive -ErrorAction Stop
    Add-ScriptLog -Color Cyan -Msg "Connected to SharePoint Admin Center"

    $collSiteCollections = Get-PnPTenantSite -ErrorAction Stop | Where-Object { $_.Title -notlike "" -and $_.Template -notlike "*Redirect*" }
    Add-ScriptLog -Color Cyan -Msg "Collected Site Collections: $($collSiteCollections.count)"
}
catch {
    Add-ScriptLog -Color Red -Msg "Error: $($_.Exception.Message)"
    break
}

$ItemCounter = 0
ForEach($oSite in $collSiteCollections) {

    $PercentComplete = [math]::Round($ItemCounter/$collSiteCollections.Count * 100, 2)
    Add-ScriptLog -Color Yellow -Msg "$($PercentComplete)% Completed - $($oSite.Url)"
    $ItemCounter++

    Try {
        Set-PnPTenantSite -Url $oSite.Url -Owners $SiteCollAdmin -ErrorAction Stop

        Connect-PnPOnline -Url $oSite.Url -ClientId $ClientId -Interactive -ErrorAction Stop
        $Admins = Get-PnPSiteCollectionAdmin

        ForEach($Admin in $Admins){

            If($Admin.PrincipalType -eq 'SecurityGroup') {      
                $collAdmins = Get-SGUsers -Site $oSite -Group $Admin
                Add-ReportRecord  -Site $oSite -AccessType "Security Group $($Group.Title)" -UserEmail $collAdmins
            }

            If($Admin.PrincipalType -eq 'User') {
                Add-ReportRecord  -Site $oSite -AccessType "Direct Permission" -UserEmail $Admin.Email
            }
        }

        Remove-PnPSiteCollectionAdmin -Owners $SiteCollAdmin
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

## Using PnP: Get only Primary Site Collection Administrators

```powershell
#################################################################
# DEFINE PARAMETERS FOR THE CASE
#################################################################
$AdminSiteURL= "https://<Domain>-admin.sharepoint.com"



#################################################################
# REPORT AND LOGS FUNCTIONS
#################################################################

function Add-ReportRecord {
    param (
        $SiteUrl,
        $Owners,
        $Remarks
    )

    $Record = New-Object PSObject -Property ([ordered]@{
        SiteUrl = $SiteUrl
        Owners = $Owners
        Remarks = $Remarks
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
$FolderPath = "$Env:USERPROFILE\Documents\"
$Date = Get-Date -Format "yyyyMMddHHmmss"
$ReportName = "SitesAdminsReport"
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
    Connect-PnPOnline -Url $AdminSiteURL -Interactive -ErrorAction Stop
    Add-ScriptLog -Color Cyan -Msg "Connected to SharePoint Online"

    $collSiteCollections = Get-PnPTenantSite | Where-Object{ ($_.Title -notlike "" -and $_.Template -notlike "*Redirect*") }
    Add-ScriptLog -Color Cyan -Msg "Collected Items: $($collSiteCollections.Count)"
}
catch {
    Add-ScriptLog -Color Red -Msg "Error message: '$($_.Exception.Message)'"
    Add-ScriptLog -Color Red -Msg "Error trace: '$($_.Exception.ScriptLineNumber)'"
    break
}

$ItemCounter = 0
ForEach($oSite in $collSiteCollections) {
       
    $PercentComplete = [math]::Round($ItemCounter/$collSiteCollections.Count * 100, 2)
    Add-ScriptLog -Color Yellow -Msg "$($PercentComplete)% Completed - Processing Item '$($oSite.URL)'"
    $ItemCounter++

    $Remarks = ""
    Try {

        If($oSite.GroupId -notlike "00000000-0000-0000-0000-000000000000") {
            try {
                $GroupOwners = (Get-PnPMicrosoft365GroupOwners -Identity ($oSite.GroupId)  | Select-Object -ExpandProperty Email) -join "; "
            }
            catch{
                $GroupOwners = "Group does not exist in Azure AD"
                $Remarks = "Group does not exist in Azure AD"
            }
        }
        elseif($Site.OwnerLoginName -like "*c:0t.c|tenant|*") {
            try{
                $GroupOwners = (Get-PnPAzureADGroup -Identity ($oSite.Owner)  | Select-Object -ExpandProperty Email) -join "; "
            }
            catch {
                $GroupOwners = "'$($oSite.OwnerName)' Security group"
                $Remarks = "Group does not exist in Azure AD"
            }
        }
        Else {
            $GroupOwners = $oSite.Owner
        }
        Add-ReportRecord -SiteUrl $oSite.Url -Owners $GroupOwners -Remarks $Remarks
    }
    Catch {
        Add-ScriptLog -Color Red -Msg "Error while processing Site Collection '$($oSite.Url)'"
        Add-ScriptLog -Color Red -Msg "Error message: '$($_.Exception.Message)'"
        Add-ScriptLog -Color Red -Msg "Error Script Line: '$($_.InvocationInfo.ScriptLineNumber)'"
        Add-ReportRecord -SiteUrl $oSite.Url -Remarks $_.Exception.Message
    }
}

Add-ScriptLog -Color Cyan -Msg "100% Completed - Finished running script"
Add-ScriptLog -Color Cyan -Msg "Report generated at at $($ReportOutput)"
```

<br>

## Using SharePoint Online Management Shell: Get All Admins using

```powershell
#################################################################
# DEFINE PARAMETERS FOR THE CASE
#################################################################
$AdminSiteURL= "https://<Domain>-admin.sharepoint.com"



#################################################################
# REPORT AND LOGS FUNCTIONS
#################################################################

Function Add-ReportRecord {
    param (
        $Site,
        $OwnerQty = "",
        $AccountType = "",
        $UserPrincipalName = "",
        $Remarks = ""
    )

    $Record = New-Object PSObject -Property ([ordered]@{
        SiteName = $Site.Title
        SiteURL = $Site.Url
        OwnerQty = $OwnerQty
        AccountType = $AccountType
        UserPrincipalName = $UserPrincipalName
        Remarks = $Remarks
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
$ReportName = "SiteCollectionAdmins"
$FolderName = $Date + "_" + $ReportName
New-Item -Path $FolderPath -Name $FolderName -ItemType "directory"

# Files
$ReportOutput = $FolderPath + $FolderName + "\" + $FolderName + "_report.csv"
$LogsOutput = $FolderPath + $FolderName + "\" + $FolderName + "_Logs.txt"

Add-ScriptLog -Color Cyan -Msg "Report will be generated at $($ReportOutput)"



#################################################################
# SCRIPT LOGIC
#################################################################

Function Get-ObjectID {
    param (
        $LoginName
    )

    $GroupID = $LoginName
    $GroupID = $GroupID -replace ('_o', '')
    $GroupID = $GroupID -replace ('c:0o.c|federateddirectoryclaimprovider|', '')
    $GroupID = $GroupID -replace ('c:0t.c|tenant|', '')
    $GroupID = $GroupID.Trim('|')

    Return $GroupID
}

Function Get-GroupUsers {
    param (
        $LoginName
    )

    If($LoginName -clike '*_o'){
        $GroupID = Get-ObjectID -LoginName $LoginName
        $GroupUsers = Get-AzureADGroupOwner -ObjectId $GroupID
    }
    Else{
        $GroupID = Get-ObjectID -LoginName $LoginName
        $GroupUsers = Get-AzureADGroupMember  -ObjectId $GroupID
    }

    Return $GroupUsers 
}

Function Get-Admins {
    param (
        $Site
    )

    $OwnerQty = "Single Owner"

    $SiteAdmins = Get-SPOUser -Site $Site.Url -Limit ALL | Where-Object { $_.IsSiteAdmin -eq $True -and $_.DisplayName -notlike "Global Administrator" -and $_.DisplayName -notlike "SharePoint Administrator" }

    If ($SiteAdmins.Length -eq 0) {
        Add-ReportRecord -Site $Site -Remarks "NO ADMIN"
        return
    }

    If ($SiteAdmins.Length -ne 1) {$OwnerQty = "Multiple Owners"}

    ForEach($Admin in $SiteAdmins) {

        If ($Admin.LoginName -clike "*@*") {

            Add-ReportRecord -Site $Site -OwnerQty $OwnerQty -AccountType "User" -UserPrincipalName $Admin.LoginName
        }
        Else {

            Try{
                $collGroupUsers = Get-GroupUsers -LoginName $Admin.LoginName

                If($collGroupUsers.Count -ne 1) {$OwnerQty = "Multiple Owners"}

                ForEach ($oUser in $collGroupUsers) {
                    Add-ReportRecord -Site $Site -OwnerQty $OwnerQty -AccountType "Security Group '$($Admin.DisplayName)'" -UserPrincipalName $oUser.UserPrincipalName
                }
            }
            Catch {

                Add-ReportRecord -Site $Site -AccountType "Security Group '$($Admin.DisplayName)'" -Remarks "DELETED GROUP"
            }
        }
    }
}


try {

    Connect-SPOService -Url $AdminSiteURL -ErrorAction Stop
    Add-ScriptLog -Color Cyan -Msg "Connected to SharePoint Online"

    Connect-AzureAD
    Add-ScriptLog -Color Cyan -Msg "Connected to Azure AD"

    $collSiteCollections = Get-SPOSite -Limit ALL -IncludePersonalSite $False | Where-Object{ ($_.Title -notlike "" -and $_.Template -notlike "*Redirect*" -and $_.Url -notlike "*my.sharepoint.com*") }
    Add-ScriptLog -Color Cyan -Msg "Site Collections: $($collSiteCollections.Count)"
}
catch {

    Add-ScriptLog -Color Red -Msg "Error message: '$($_.Exception.Message)'"
    Add-ScriptLog -Color Red -Msg "Error trace: '$($_.Exception.ScriptStackTrace)'"
    break
}

$ItemCounter = 0
ForEach($oSiteCollection in $collSiteCollections) {

    $PercentComplete = [math]::Round($ItemCounter/$collSiteCollections.Count * 100, 2)
    Add-ScriptLog -Color Yellow -Msg "$($PercentComplete)% Completed - Processing Site Collection: $($oSiteCollection.URL)"
    $ItemCounter++

    Try {

        Get-Admins -Site $oSiteCollection
    }
    Catch {

        Add-ScriptLog -Color Red -Msg "Error while processing Site Collection '$($oSiteCollection.Url)'"
        Add-ScriptLog -Color Red -Msg "Error message: '$($_.Exception.Message)'"
        Add-ScriptLog -Color Red -Msg "Error trace: '$($_.Exception.ScriptStackTrace)'"
    }
}

if($collSiteCollections.Count -ne 0) { 

    $PercentComplete = [math]::Round($ItemCounter/$collSiteCollections.Count * 100, 1) 
    Add-ScriptLog -Color Cyan -Msg "$($PercentComplete)% Completed - Finished running script"
}
Add-ScriptLog -Color Cyan -Msg "Report generated at at $($ReportOutput)"
```