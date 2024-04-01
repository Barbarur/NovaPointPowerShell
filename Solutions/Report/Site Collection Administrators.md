#Report #SharePointOnline #OneDrive #PowerShell #PnP #SPOService #SiteCollection #SiteAdmin 

<br>

## Using PnP: Get Primary Admins

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

## Using PnP: Get All Site Collection Admins and Subsite Owners

```powershell
#Define Parameters
$AdminSiteURL= "https://<Domain>-admin.sharepoint.com"
$ReportOutput = "C:\AdminOwnerPermissions.csv"


#Get Credentials to connect
$Cred  = Get-Credential

#Connect to Services
Connect-PnPOnline -Url $AdminSiteURL –Credential $Cred
Connect-AzureAD –Credential $Cred

#Get owners of each Site
$Global:Results = @()
$ItemCounter = 0 


#Add records to the Report
Function Add-Report($UserName, $UserEmail, $AccessType, $GroupName, $AccountType, $AccountName, $SitePermissionLevels){
    $Global:Results += New-Object PSObject -Property ([ordered]@{
        SiteName               = $Site.Title
        SiteURL                = $Site.url
        UserName               = $UserName
        UserEmail              = $UserEmail
        AccessType             = $AccessType
        GroupName              = $GroupName
        AccountType            = $AccountType
        AccountName            = $AccountName
        PermissionLevel        = $SitePermissionLevels
    })
}


# Clean up LoginName to get Group ID
Function Get-ObjectID($LoginName){
    $GroupID = $LoginName
    $GroupID = $GroupID -replace ('_o', '')
    $GroupID = $GroupID -replace ('c:0o.c|federateddirectoryclaimprovider|', '')
    $GroupID = $GroupID -replace ('c:0t.c|tenant|', '')
    $GroupID = $GroupID.Trim('|')

    Return $GroupID
}


# Function to get Security Group Users depending if Added group is owners or only members
Function Get-GroupUsers($LoginName){
    # Get Group Owners
    Try{
        If($LoginName -clike '*_o'){
            $GroupID = Get-ObjectID -LoginName $LoginName
            $GroupUsers = Get-AzureADGroupOwner -ObjectId $GroupID
        }
        # Get Group Members
        Else{
            $GroupID = Get-ObjectID -LoginName $LoginName
            $GroupUsers = Get-AzureADGroupMember  -ObjectId $GroupID
        }
    }
    Catch{
        Clear-Variable -Name GroupUsers
    }
    Return $GroupUsers 
}



#Get all Sites and iterate
$Sites = Get-PnPTenantSite | Where{ ($_.Title -notlike "" -and $_.Template -notlike "*Redirect*" -and $_.Url -notlike "*my.sharepoint.com*") }
Write-Host -f Cyan "Total number of Sites:"$Sites.Count
ForEach($Site in $Sites){

    #Status notification
    $ItemCounter++
    $ItemProcess = [math]::Round($ItemCounter/$Sites.Count*100,1)
    Write-Progress -PercentComplete $ItemProcess -Activity "Processing $($ItemProcess)%" -Status "Site '$($Site.Url)"
    Write-Host -f Yellow $Site.url

    Connect-PnPOnline -Url $Site.url –Credential $Cred
    
    ####################################
    # CHECK SITE COLLECTIONS ADMINS
    ####################################
    $Admins = Get-PnPSiteCollectionAdmin
    ForEach($Admin in $Admins){

        # SECURITY GROUP
        If($Admin.PrincipalType -eq 'SecurityGroup'){
            Write-Host 'Checking Admin  ##  Security Group  ## '$Admin.Email' ## '$Admin.LoginName

            $GroupUsers = Get-GroupUsers -LoginName $Admin.LoginName
            
            ForEach($GroupUser in $GroupUsers){
                Add-Report -UserName $GroupUser.DisplayName -UserEmail $GroupUser.UserPrincipalName -AccessType "Direct Access" -GroupName '' -AccountType 'Security Group' -AccountName $Admin.Title -SitePermissionLevels "Admin"
            }
        }

        # USER
        If($Admin.PrincipalType -eq 'User'){
            Write-Host 'Checking Admin  ##  User  ##'$Admin.Email' ## '$Admin.LoginName
            Add-Report -UserName $Admin.Title -UserEmail $Admin.Email -AccessType "Direct Access" -GroupName '' -AccountType "User" -AccountName $Admin.Title -SitePermissionLevels "Admin"
        }
    }



    ####################################
    # CHECK SUB-SITES FULL CONTROL USERS
    ####################################
    $SubSites = Get-PnPSubWeb -Recurse -Includes HasUniqueRoleAssignments
    ForEach($Site in $SubSites){
        Write-Host -f Yellow $Site.url

        # SUBSITE WITH UNIQUE PERMISSIONS
        if ($Site.HasUniqueRoleAssignments){
            Connect-PnPOnline -Url $Site.url –Credential $Cred

            $WebRoles = Get-PnPWeb -Includes RoleAssignments
            ForEach ($SiteRoleAssignment in $WebRoles.RoleAssignments){
                #Get the Permission Levels assigned and Member
                Get-PnPProperty -ClientObject $SiteRoleAssignment -Property RoleDefinitionBindings, Member

                #Get the Permission Levels assigned
                $SitePermissionLevels = $SiteRoleAssignment.RoleDefinitionBindings | Where { ($_.Name -eq "Full Control")}
                If($SitePermissionLevels.Length -eq 0 -or $SiteRoleAssignment.Member.Title -clike '*Limited Access System Group*') {Continue}

                $SitePermissionType = $SiteRoleAssignment.Member.PrincipalType

                # Check if user is in SharePoint Group
                If($SitePermissionType -eq "SharePointGroup") {
                    
                    $GroupMembers = Get-PnPGroupMember -Identity $SiteRoleAssignment.Member.Title
                    ForEach($GroupMember in $GroupMembers){
                        
                        If($GroupMember.Title -eq "Everyone" -or $GroupMember.Title -eq "Everyone except external users"){Continue}

                        If($GroupMember.PrincipalType -eq "SecurityGroup"){
                            Write-Host -f Cyan 'Checking SharePoint Group'$SiteRoleAssignment.Member.Title' ##  Security Group  ## '$GroupMember.Title' ## '$GroupMember.LoginName
                    
                            $GroupUsers = Get-GroupUsers -LoginName $GroupMember.LoginName
                    
                            ForEach($GroupUser in $GroupUsers){
                                Add-Report -UserName $GroupUser.DisplayName -UserEmail $GroupUser.UserPrincipalName -AccessType "Direct Access" -GroupName '' -AccountType 'Security Group' -AccountName $Admin.Title -SitePermissionLevels "Full Control"
                            }
                        }
                        Else{
                            Write-Host -f Cyan 'Checking SharePoint Group '$SiteRoleAssignment.Member.Title' ##  User  ## '$GroupMember.Title' ## '$GroupMember.LoginName
                            Add-Report -UserName $GroupMember.Title -UserEmail $GroupMember.Email -AccessType 'SharePoint Group' -GroupName $SiteRoleAssignment.Member.Title -AccountType "User" -AccountName $GroupMember.Title -SitePermissionLevels "Full Control"
                        }
                    }
                }
            }
        }

        # NO UNIQUE PERMISSIONS SUBSITE
        Else{
            Add-Report -UserName 'Same as parent site' -UserEmail 'Same as parent site' -AccessType 'Same as parent site' -GroupName 'Same as parent site' -AccountType 'Same as parent site' -AccountName 'Same as parent site' -SitePermissionLevels 'Same as parent site'
        }
    }
}
#Close status notification
Write-Progress -Activity "Processing $($ItemProcess)%" -Status "Site '$($Site.URL)"

If($Global:Results.count -eq 0){
    Write-host -b Green "Report is empty!"
}
Else{
    #Export the results to CSV
    If (Test-Path $ReportOutput) { Remove-Item $ReportOutput }
    $Global:Results | Export-Csv -Path $ReportOutput -NoTypeInformation
    Write-host -b Green "Report Generated Successfully!"
    Write-host -f Green $ReportOutput
}
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