#Report #AzureAD #CSV #Permissions #PnP #PowerShell #SharePointOnline 

<br>

## Option 1: User access across all Site Collections and Subsites

This script doesn’t actually review the permissions, only if the user is registered to the site. Which means the user has access or had previously access to the site.

```powershell
################################################################
# PARAMETERS TO BE CHANGED TO MATCH CURRENT CASE
################################################################
$AdminSiteURL = "https://<DOMAIN>-admin.sharepoint.com"
$SiteCollAdmin = "<USER@EMAIL.com>"
$CheckUserEmail = "<USER@EMAIL.com>"



################################################################
# REPORT AND LOGS FUNCTIONS
################################################################
# Add new record on the report
Function Add-ReportRecord($SiteURL) {
    $Record = New-Object PSObject -Property ([ordered]@{
        "Site URL"          = $SiteURL
        "User Email"        = $CheckUserEmail
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
$Date = Get-Date -Format "yyyyMMdd_HHmmss"
$ReportName = "UserAccessReport"
$FolderName = $Date + "_" + $ReportName
$FolderPath = "$Env:USERPROFILE\Documents\"
New-Item -Path $FolderPath -Name $FolderName -ItemType "directory"
$ReportOutput = $FolderPath + $FolderName + "\" + $ReportName + ".csv"

# Create logs file
$LogsName = $ReportName + "_Logs.txt"
$LogsOutput = $FolderPath + $FolderName + "\" + $LogsName

Add-ScriptLog -Color Cyan -Msg "Report will be generated at $($ReportOutput)"



#################################################################
# SCRIPT LOGIC
#################################################################
Function Find-UserAccess($SiteURL) {
    Connect-PnPOnline -Url $SiteURL -Interactive

    $UsersList = Get-PnPUser | Where-Object { $_.Email -eq $CheckUserEmail}

    If ($UsersList.Length -eq 0){continue}

    Add-ScriptLog -Color Cyan -Msg "User found in Site '$($SiteURL)'"
    Add-ReportRecord -SiteURL $SiteURL
}

# Connect to SharePoint Site and collect subsites
try {
    Connect-PnPOnline -Url $AdminSiteURL -Interactive -ErrorAction Stop
    Add-ScriptLog -Color Cyan -Msg "Connected to SharePoint Admin Center"
    $SitesList = Get-PnPTenantSite -ErrorAction Stop | Where-Object{ ($_.Title -notlike "" -and $_.Template -notlike "*Redirect*") }
    Add-ScriptLog -Color Cyan -Msg "Collected Site Collections: $($SitesList.count)"
}
catch {
    Add-ScriptLog -Color Red -Msg "Error: $($_.Exception.Message)"
}

# Itinerate across all SitesList
$ItemCounter = 0
ForEach($Site in $SitesList) {
    # Adding notification and logs
    $PercentComplete = [math]::Round($ItemCounter/$SitesList.Count * 100, 2)
    Add-ScriptLog -Color Yellow -Msg "$($PercentComplete)% Completed - Checking Site: $($Site.Title)"
    $ItemCounter++

    Try {
        Set-PnPTenantSite -Url $Site.Url -Owners $SiteCollAdmin -ErrorAction Stop
        
        Find-UserAccess -SiteURL $Site.Url

        $SubSites = Get-PnPSubWeb -Recurse
        ForEach($Site in $SubSites) {
            Find-UserAccess -SiteURL $Site.Url
        }
    }
    Catch {
        Add-ScriptLog -Color Red -Msg "Error adding user as Site Collection Administrator for Site Collection '$($Site.Title)'"
        Add-ReportRecord -SiteURL $Site.Url -Remarks $_.Exception.Message
    }

    Remove-PnPSiteCollectionAdmin -Owners $SiteCollAdmin
}
# Close status notification
$PercentComplete = [math]::Round($ItemCounter/$SitesList.Count * 100, 1)
Add-ScriptLog -Color Cyan -Msg "$($PercentComplete)% Completed - Finished running script"
Add-ScriptLog -Color Cyan -Msg "Report generated at at $($ReportOutput)"
```

<br>

## Option 2: User list access across all Site Collections

It requires a csv file with a column with header Email which includes the list of target users.

```powershell
# Define Parameters
$AdminSiteURL = "https://DOMAIN-admin.sharepoint.com"
$SiteCollAdmin = "XXX@XXX.com"
$UserListCSV = "$Env:USERPROFILE\Desktop\UserList.csv"
$ReportOutput = "$Env:USERPROFILE\Desktop\SelectedUsersAccessReport.csv"

# Script variables
$Global:Results = @()
$ItemCounter = 0

Connect-SPOService -Url $AdminSiteURL

$CSVData = Import-CSV $UserListCSV

$HuntingList = @()
ForEach($Row in $CSVData)
{

    $HuntingList += $Row.Email

}

write-host -f Yellow $HuntingList

# Get all Site collections and iterate
$SitesList = Get-SPOSite -Limit ALL -IncludePersonalSite $False | Where{ ($_.Title -notlike "" -and $_.Template -notlike "*Redirect*") }
Write-Host -f Cyan 'Total number of Site Collections:'$SitesList.Count
ForEach($Site in $SitesList)
{
    
    # Status notification
    $ItemCounter++
    $PercentComplete = [math]::Round($ItemCounter/$SitesList.Count*100,1)
    Write-Progress -PercentComplete $PercentComplete -Activity "Processing $($PercentComplete)%" -Status "Site '$($Site.URL)"
    Write-Host -f Yellow $Site.url

    # Check if site contains external users and add to the report
    Start-Sleep -Seconds 2
    Try{
        
        Set-SPOUser -Site $Site.Url -LoginName $SiteCollAdmin -IsSiteCollectionAdmin $True
        
        $UsersList = Get-SPOUser -Site $Site.Url -Limit ALL | Where{$_.LoginName -in $HuntingList}
        
        if($UsersList.count -ne 0)
        {
            ForEach($User in $UsersList)
            {

                $Global:Results += New-Object PSObject -Property ([ordered]@{
                    SiteName               = $Site.Title
                    SiteURL                = $Site.url
                    UserName               = $User.DisplayName
                    UserEmail              = $User.LoginName
                })
        
            }
        }
        Else
        {
            
            $Global:Results += New-Object PSObject -Property ([ordered]@{
                SiteName               = $Site.Title
                SiteURL                = $Site.url
                UserName               = "No Match"
                UserEmail              = "No Match"
            })

        }

    }
    
    Catch
    {

        Write-Host -f Red $Site.url"ERROR while checking external users!"
    
    }

    Set-SPOUser -Site $Site.Url -LoginName $SiteCollAdmin -IsSiteCollectionAdmin $False

}

# Close status notification
Write-Progress -Activity "Processing $($PercentComplete)%" -Status "Site '$($Site.URL)"


If($Global:Results.count -eq 0){
    Write-host -b Red "Report is empty!"
}
Else{
    #Export the results to CSV
    If (Test-Path $ReportOutput) { Remove-Item $ReportOutput }
    $Global:Results | Export-Csv -Path $ReportOutput -NoTypeInformation
    Write-Host -b Green "Report is ready!"
    Write-host -f Green $ReportOutput
}
```

<br>

## Option 3: User Admin permissions across all Site Collections

```powershell
#Define Parameters
$AdminSiteURL= "https://DOMAIN-admin.sharepoint.com"
$ReportOutput = "C:\UserSitePermissions.csv"
$CheckUser = "USER@gEMAIL.com"

#Get Credentials to connect
$Cred  = Get-Credential

#Connect to Services
Connect-PnPOnline -Url $AdminSiteURL –Credential $Cred
Connect-AzureAD –Credential $Cred

#Get owners of each Site
$Global:Results = @()
$ItemCounter = 0 


#Add records to the Report
Function Add-Report($AccessType, $GroupName, $AccountType, $AccountName, $SitePermissionLevels){
    $Global:Results += New-Object PSObject -Property ([ordered]@{
        SiteName               = $TenantSite.Title
        SiteURL                = $TenantSite.Url
        UserEmail              = $CheckUser
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
            $GroupUsers = Get-AzureADGroupOwner -ObjectId $GroupID | Where {$_.UserPrincipalName -eq $CheckUser}
        }
        # Get Group Members
        Else{
            $GroupID = Get-ObjectID -LoginName $LoginName
            $GroupUsers = Get-AzureADGroupMember  -ObjectId $GroupID | Where {$_.UserPrincipalName -eq $CheckUser}
        }
    }
    Catch{
        Clear-Variable -Name GroupUsers
    }
    Return $GroupUsers 
}


#Get all Sites and iterate
$TenantSites = Get-PnPTenantSite | Where{ ($_.Title -notlike "" -and $_.Template -notlike "*Redirect*" -and $_.Url -notlike "*my.sharepoint.com*") }
Write-Host -f Cyan $TenantSites.Count
ForEach($TenantSite in $TenantSites){

    #Status notification
    $ItemCounter++
    $ItemProcess = [math]::Round($ItemCounter/$TenantSites.Count*100,1)
    Write-Progress -PercentComplete $ItemProcess -Activity "Processing $($ItemProcess)%" -Status "Site '$($TenantSite.Url)"
    Write-Host -f Yellow $TenantSite.url

    Connect-PnPOnline -Url $TenantSite.url –Credential $Cred
    
    ####################################
    # CHECK USER IS SITE ADMIN
    ####################################
    $Admins = Get-PnPSiteCollectionAdmin
    ForEach($Admin in $Admins){

        # SECURITY GROUP
        If($Admin.PrincipalType -eq 'SecurityGroup'){
            Write-Host 'Checking Admin  ##  Security Group  ## '$Admin.Email' ## '$Admin.LoginName

            $GroupUsers = Get-GroupUsers -LoginName $Admin.LoginName
            
            If($GroupUsers.count -ne 0){
                Add-Report -AccessType "Direct Access" -GroupName '' -AccountType 'Security Group' -AccountName $Admin.Title -SitePermissionLevels "Admin"
            } 
        }

        # USER
        If($Admin.PrincipalType -eq 'User'){
            Write-Host 'Checking Admin  ##  User  ##'$Admin.Email' ## '$Admin.LoginName
            
            If($Admin.Email -eq $CheckUser){
                Add-Report -AccessType "Direct Access" -GroupName '' -AccountType "User" -AccountName $Admin.Title -SitePermissionLevels "Admin"
            }
        }
    }
}
#Close status notification
Write-Progress -Activity "Processing $($ItemProcess)%" -Status "Site '$($TenantSite.URL)"

If($Global:Results.count -eq 0){
    Write-host -b Red "Report is empty!"
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

## Option 4: User direct permissions across all Site Collections

Due the high workload of iterate cross all Site Collections, this script doesn’t check unique permissions inside each site.

```powershell
#Define Parameters
$AdminSiteURL= "https://DOMAIN-admin.sharepoint.com"
$ReportOutput = "C:\UserSitePermissions.csv"
$CheckUser = "USER@gEMAIL.com"

#Get Credentials to connect
$Cred  = Get-Credential

#Connect to Services
Connect-PnPOnline -Url $AdminSiteURL –Credential $Cred
Connect-AzureAD –Credential $Cred

#Get owners of each Site
$Global:Results = @()
$ItemCounter = 0 


#Add records to the Report
Function Add-Report($AccessType, $GroupName, $AccountType, $AccountName, $SitePermissionLevels){
    $Global:Results += New-Object PSObject -Property ([ordered]@{
        SiteName               = $TenantSite.Title
        SiteURL                = $TenantSite.Url
        UserEmail              = $CheckUser
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
            $GroupUsers = Get-AzureADGroupOwner -ObjectId $GroupID | Where {$_.UserPrincipalName -eq $CheckUser}
        }
        # Get Group Members
        Else{
            $GroupID = Get-ObjectID -LoginName $LoginName
            $GroupUsers = Get-AzureADGroupMember  -ObjectId $GroupID | Where {$_.UserPrincipalName -eq $CheckUser}
        }
    }
    Catch{
        Clear-Variable -Name GroupUsers
    }
    Return $GroupUsers 
}


#Get all Sites and iterate
$TenantSites = Get-PnPTenantSite | Where{ ($_.Title -notlike "" -and $_.Template -notlike "*Redirect*" -and $_.Url -notlike "*my.sharepoint.com*") }
Write-Host -f Cyan $TenantSites.Count
ForEach($TenantSite in $TenantSites){

    #Status notification
    $ItemCounter++
    $ItemProcess = [math]::Round($ItemCounter/$TenantSites.Count*100,1)
    Write-Progress -PercentComplete $ItemProcess -Activity "Processing $($ItemProcess)%" -Status "Site '$($TenantSite.Url)"
    Write-Host -f Yellow $TenantSite.url

    Connect-PnPOnline -Url $TenantSite.url –Credential $Cred
    
    ####################################
    # CHECK USER IS SITE ADMIN
    ####################################
    $Admins = Get-PnPSiteCollectionAdmin
    ForEach($Admin in $Admins){

        # SECURITY GROUP
        If($Admin.PrincipalType -eq 'SecurityGroup'){
            Write-Host 'Checking Admin  ##  Security Group  ## '$Admin.Email' ## '$Admin.LoginName

            $GroupUsers = Get-GroupUsers -LoginName $Admin.LoginName
            
            If($GroupUsers.count -ne 0){
                Add-Report -AccessType "Direct Access" -GroupName '' -AccountType 'Security Group' -AccountName $Admin.Title -SitePermissionLevels "Admin"
            } 
        }

        # USER
        If($Admin.PrincipalType -eq 'User'){
            Write-Host 'Checking Admin  ##  User  ##'$Admin.Email' ## '$Admin.LoginName
            
            If($Admin.Email -eq $CheckUser){
                Add-Report -AccessType "Direct Access" -GroupName '' -AccountType "User" -AccountName $Admin.Title -SitePermissionLevels "Admin"
            }
        }
    }


    ####################################
    # CHECK USER PERMISSIONS ON THE SITE
    ####################################
    
    #Get Assigned permissions and itinerate
    $WebRoles = Get-PnPWeb -Includes RoleAssignments
    ForEach ($SiteRoleAssignment in $WebRoles.RoleAssignments){
        
        #Get the Permission Levels assigned and Member
        Get-PnPProperty -ClientObject $SiteRoleAssignment -Property RoleDefinitionBindings, Member

        #Get the Permission Levels assigned
        $SitePermissionLevels = ($SiteRoleAssignment.RoleDefinitionBindings | Select -ExpandProperty Name | Where { ($_ -ne "Limited Access") -and ($_ -ne "Web-Only Limited Access")} ) -join ","
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
                    
                    If($GroupUsers.count -ne 0){
                        Add-Report -AccessType 'SharePoint Group' -GroupName $SiteRoleAssignment.Member.Title -AccountType 'Security Group' -AccountName $GroupMember.Title -SitePermissionLevels $SitePermissionLevels
                    }
                }
                Else{
                    Write-Host -f Cyan 'Checking SharePoint Group '$SiteRoleAssignment.Member.Title' ##  User  ## '$GroupMember.Title' ## '$GroupMember.LoginName
                    If($GroupMember.Email -eq $CheckUser){
                        Add-Report -AccessType 'SharePoint Group' -GroupName $SiteRoleAssignment.Member.Title -AccountType "User" -AccountName $GroupMember.Title -SitePermissionLevels $SitePermissionLevels
                    }
                }
            }
        }

        # Check if user has direct access
        Else{
            
            If($SiteRoleAssignment.Member.Title -eq "Everyone" -or $SiteRoleAssignment.Member.Title -eq "Everyone except external users"){Continue}

            If($SiteRoleAssignment.Member.PrincipalType -eq "SecurityGroup"){
                Write-Host -f Magenta 'Checking Direct Access  ##  Security Group  ## '$SiteRoleAssignment.Member.Title' ## '$SiteRoleAssignment.Member.LoginName
                
                $GroupUsers = Get-GroupUsers -LoginName $SiteRoleAssignment.Member.LoginName
                
                If($GroupUsers.count -ne 0){
                    Add-Report -AccessType "Direct Access" -GroupName '' -AccountType 'Security Group' -AccountName $SiteRoleAssignment.Member.Title -SitePermissionLevels $SitePermissionLevels
                }
            }
            Else{
                Write-Host -f Magenta 'Checking Direct Access  ##  User  ## '$SiteRoleAssignment.Member.Title' ## '$SiteRoleAssignment.Member.LoginName
                If($SiteRoleAssignment.Member.Title -eq $CheckUser){
                    Add-Report -AccessType "Direct Access" -GroupName '' -AccountType "User" -AccountName -SitePermissionLevels $SitePermissionLevels
                }
            }
        }
    }
}
#Close status notification
Write-Progress -Activity "Processing $($ItemProcess)%" -Status "Site '$($TenantSite.URL)"

If($Global:Results.count -eq 0){
    Write-host -b Red "Report is empty!"
}
Else{
    #Export the results to CSV
    If (Test-Path $ReportOutput) { Remove-Item $ReportOutput }
    $Global:Results | Export-Csv -Path $ReportOutput -NoTypeInformation
    Write-host -b Green "Report Generated Successfully!"
    Write-host -f Green $ReportOutput
}
```