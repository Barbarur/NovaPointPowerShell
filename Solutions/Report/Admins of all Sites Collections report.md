#Report #SharePointOnline #OneDrive #PowerShell #Pnp #SPOService #SiteCollection #SiteAdmin 

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
        $Site,
        $OwnerType,
        $OwnerQty,
        $User
    )

    $Record = New-Object PSObject -Property ([ordered]@{
        SiteName               = $Site.Title
        SiteURL                = $Site.Url
        OwnerType              = $OwnerType
        OwnerQty               = $OwnerQty
        OwnerName              = $User.DisplayName
        OwnerEmail             = $User.Mail
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
$FolderPath = "$Env:USERPROFILE\Documents\SPOScripts\"
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

    Connect-AzureAD -Interactive -ErrorAction Stop
    Add-ScriptLog -Color Cyan -Msg "Connected to Azure AD"

    $collSiteCollections = Get-PnPTenantSite | Where-Object{ ($_.Title -notlike "" -and $_.Template -notlike "*Redirect*") }
    Add-ScriptLog -Color Cyan -Msg "Collected Items: $($collSiteCollections.Count)"
}
catch {
    Add-ScriptLog -Color Red -Msg "Error: $($_.Exception.Message)"
    break
}

$ItemCounter = 0
ForEach($oSite in $collSiteCollections) {
       
    $PercentComplete = [math]::Round($ItemCounter/$collSiteCollections.Count * 100, 2)
    Add-ScriptLog -Color Yellow -Msg "$($PercentComplete)% Completed - Processing Item '$($oSite.URL)'"
    $ItemCounter++

    
    $OwnerQty = "Single Owner"

    If($Site.GroupID -notLike "00000000-0000-0000-0000-000000000000") {
        Try {
            $Owners = Get-AzureADGroupOwner -ObjectId $Site.GroupId

            If($Owners.Count -ne 1) {$OwnerQty = "Multiple Owners"}

            ForEach($Owner in $Owners.UserPrincipalName) {
                $User = Get-AzureADUser -ObjectId $Owner
                Add-ReportRecord -Site $oSite -OwnerType "MS365 Group" -OwnerQty $OwnerQty -User $User
            }
        }
        Catch {
            Add-ReportRecord -Site $oSite -OwnerType "MS365 Group" -OwnerQty "DELETED GROUP"
        }
    }
    Else {
        If($Site.Owner.Length -eq 0) {
            Add-ReportRecord -Site $oSite -OwnerType "User" -OwnerQty "No Owner" -User ""
        }
        Else {
            Try {
                $User = Get-AzureADUser -ObjectId $Site.Owner
                Add-ReportRecord -Site $oSite -OwnerType "User" -OwnerQty $OwnerQty -User $User
            }
            Catch {
                Add-ReportRecord -Site $oSite -OwnerType "User" -OwnerQty "DELETED GROUP"
            }
        }
    }
}

if($collSiteCollections.Count -ne 0) { 
    $PercentComplete = [math]::Round($ItemCounter/$collSiteCollections.Count * 100, 1) 
    Add-ScriptLog -Color Cyan -Msg "$($PercentComplete)% Completed - Finished running script"
}
Add-ScriptLog -Color Cyan -Msg "Report generated at at $($ReportOutput)"
```

<br>

## Using PnP: Get All Site Collection Admins and Subsite Owners

```powershell
#Define Parameters
$AdminSiteURL= "https://DOMAIN-admin.sharepoint.com"
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
#Define Parameters
$AdminSiteURL= "https://DOMAIN-admin.sharepoint.com"
$ReportOutput = "C:\Temp\SitesOwnersReport.csv"

#Get Credentials
$Cred = Get-Credential

#Connect to SharePoint Online Admin Center
Connect-SPOService -Url $AdminSiteURL –Credential $Cred
Connect-AzureAD –Credential $Cred

#Get owners of each Site
$Global:Results = @()
$ItemCounter = 0 

#Function to add Owners to the Report
Function Add-Report($OwnerType, $OwnerName, $OwnerEmail){
    $Global:Results += New-Object PSObject -Property ([ordered]@{
        SiteName               = $Site.Title
        SiteURL                = $Site.Url
        OwnerType              = $OwnerType
        OwnerQty               = $OwnerQty
        OwnerName              = $OwnerName
        OwnerEmail             = $OwnerEmail
        StorageTotalGB         = ('{0:N2}' -f ($Site.StorageQuota/1024))
        StorageUsedGB          = ('{0:N2}' -f ([math]::Round($Site.StorageUsageCurrent/1024,2)))
        StorageFreeGB          = ('{0:N2}' -f (($Site.StorageQuota-$Site.StorageUsageCurrent)/1024))
        })
    }

#Get all Site colections
$Sites = Get-SPOSite -Limit ALL -IncludePersonalSite $False  | Where{ ($_.Title -notlike "") }
Write-Host -f Yellow "Total Number of Sites Found: "$Sites.count

Foreach ($Site in $Sites) {
    #Status notification
    $ItemCounter++
    $ItemProcess = [math]::Round($ItemCounter/$Sites.Count*100,1)
    Write-Progress -PercentComplete $ItemProcess -Activity "Processing $($ItemProcess)%" -Status "Site '$($Site.URL)"
    $Msg = "Collecting Owners for Site '{0}'... " -f $Site.Title
    Write-Host -f Yellow $Msg -NoNewline
    
    $OwnerQty = "Single Owner"

    Try {
        #Get all Site Collection Administrators
        $SiteAdmins = Get-SPOUser -Site $Site.Url -Limit ALL | Where { $_.IsSiteAdmin -eq $True}

        If ($SiteAdmins.Length -eq 0) {
            $OwnerQty = "NO OWNER"
            Add-Report -OwnerType "User" -OwnerName "NO OWNER" -OwnerEmail ""
            Continue
            }

        If ($SiteAdmins.Length -ne 1) {$OwnerQty = $OwnerQty = "Multiple Owners"}

        ForEach($Admin in $SiteAdmins) {
            If ($Admin.LoginName -clike "*@*") {
                Try {
                    $User = Get-AzureADUser -ObjectId $Admin.LoginName
                    Add-Report -OwnerType "User" -OwnerName $User.DisplayName -OwnerEmail $User.Mail
                    }
                Catch {
                    Add-Report -OwnerType "User" -OwnerName "DELETED USER" -OwnerEmail ""
                    }
                }
            Else {
                $ObjectId = $Admin.LoginName -replace ('_o','')
                Try{
                    $Owners= Get-AzureADGroupOwner -ObjectId $ObjectId
                    If($Owners.Count -ne 1) {$OwnerQty = "Multiple Owners"}

                    ForEach ($Owner in $Owners.UserPrincipalName) {
                        $User = Get-AzureADUser -ObjectId $Owner
                        Add-Report -OwnerType "MS365 Group" -OwnerName $User.DisplayName -OwnerEmail $User.Mail 
                        }
                    }
                Catch {
                    Add-Report -OwnerType "MS365 Group" -OwnerName "DELETED GROUP" -OwnerEmail ""
                    }
                }
            }
        }
    Catch {
        $OwnerQty = "NO OWNER"
        Add-Report -OwnerType "User" -OwnerName "NO OWNER" -OwnerEmail ""
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
```