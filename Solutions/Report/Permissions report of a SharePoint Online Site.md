#Report #SharePointOnline #OneDrive #PowerShell #PnP #DocumentLibrary #ItemList #SharedLink 

<br>

## Permissions from a specific location in a Site

```powershell
#################################################################
# DEFINE PARAMETERS FOR THE CASE
#################################################################
$SiteURL= "https://<Domain>.sharepoint.com/sites/12345678"
$ListName = "Documents"
$TargetFolderPath = "*/<SiteName>/<LibraryName>/<Foldername>/*"



#################################################################
# REPORT AND LOGS FUNCTIONS
#################################################################

function Add-ReportRecord {
    param (
        $Item,
        $RoleAssignment,
        $UserName,
        $UserPrincipalName,
        $UserEmail,
        $PermissionLevels ,
        $Remarks
    )
    
    $Versions = Get-PnPProperty -ClientObject $Item -Property Versions

    $Record = New-Object PSObject -Property ([ordered]@{
        ItemName               = $Item.FieldValues["FileLeafRef"]      
        ItemRelativeURL        = $Item.FieldValues["FileRef"]
        ItemType               = $Item.FileSystemObjectType
        AccessType             = $RoleAssignment.Member.PrincipalType
        GroupName              = $RoleAssignment.Member.Title
        UserName               = $UserName
        UserPrincipalName      = $UserPrincipalName
        UserEmail              = $UserEmail
        PermissionLevels       = $PermissionLevels
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
$FolderPath = "$Env:USERPROFILE\Documents\SPOScripts\"
$Date = Get-Date -Format "yyyyMMddHHmmss"
$ReportName = "FolderPermissionsReport"
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
    Add-ScriptLog -Color Cyan -Msg "Connected to Site"

    $collItems = Get-PnPListItem -List $ListName -PageSize 3000 | Where-Object { $_["FileRef"] -like $TargetFolderPath }
    Add-ScriptLog -Color Cyan -Msg "Collected Items: $($collItems.Count)"
}
catch {
    Add-ScriptLog -Color Red -Msg "Error: $($_.Exception.Message)"
    break
}

Function Get-UserPermissions($RoleAssignments, $Item) {
    ForEach ($RoleAssignment in $RoleAssignments) {

        Get-PnPProperty -ClientObject $RoleAssignment -Property RoleDefinitionBindings, Member
        
        $PermissionLevels = ($RoleAssignment.RoleDefinitionBindings | Select -ExpandProperty Name | Where { ($_ -ne "Limited Access") -and ($_ -ne "Web-Only Limited Access")} ) -join ","
        If($PermissionLevels.Length -eq 0) {Continue}

        Write-host -f DarkYellow "Getting Unique Permissions for $($RoleAssignment.Member.Title)"

        $PermissionType = $RoleAssignment.Member.PrincipalType
        $GroupName = $RoleAssignment.Member.Title

        If($PermissionType -eq "SharePointGroup")
            {
            If ($RoleAssignment.Member.Title -like "SharingLinks*"){
                $PermissionType = "Shared Link"
                }
            $Users = Get-PnPProperty -ClientObject ($RoleAssignment.Member) -Property Users -ErrorAction SilentlyContinue
            If($Users.Count -eq 0) { Continue }
            Else {
                ForEach ($User in $Users) {
                    If($User.Title -eq "System Account") {Continue}

                    Add-ReportRecord -Item $Item -RoleAssignment $RoleAssignment -UserName $User.Title -UserPrincipalName $User.UserPrincipalName -UserEmail $User.Email -PermissionLevels $PermissionLevels

                    }
                }
            }
        Else {
            Add-ReportRecord -Item $Item -RoleAssignment $RoleAssignment -UserName $RoleAssignment.Member.Title -UserPrincipalName $RoleAssignment.Member.LoginName.Replace('i:0#.f|membership|','') -UserEmail $RoleAssignment.Member.LoginName.Replace('i:0#.f|membership|','') -PermissionLevels $PermissionLevels
        }
    }
}

$ItemCounter = 0
ForEach($oItem in $collItems) {
       
    $PercentComplete = [math]::Round($ItemCounter/$collItems.Count * 100, 2)
    Add-ScriptLog -Color Yellow -Msg "$($PercentComplete)% Completed - Processing Item '$($oItem["FileRef"])'"
    $ItemCounter++

    $ItemHasUniquePermissions = Get-PnPProperty -ClientObject $oItem -Property "HasUniqueRoleAssignments"
    If($ItemHasUniquePermissions) {
           
        $ItemRoleAssignments = Get-PnPProperty -ClientObject $oItem -Property RoleAssignments
        Get-UserPermissions -RoleAssignments $ItemRoleAssignments -Item $oItem
    }
}

$PercentComplete = [math]::Round($ItemCounter/$collItems.Count * 100, 1)
Add-ScriptLog -Color Cyan -Msg "$($PercentComplete)% Completed - Finished running script"
Add-ScriptLog -Color Cyan -Msg "Report generated at at $($ReportOutput)"
```

<br>

## User with Full Control at the Site Level

```powershell
#################################################################
# DEFINE PARAMETERS FOR THE CASE
#################################################################
$AdminSiteURL = "https://<DOMAIN>-admin.sharepoint.com" # SharePoint Admin Center Url
$SiteCollAdmin = "<ADMIN@EMAIL.com>" # Global or SharePoint Admin used for loging running the script.



#################################################################
# REPORT AND LOGS FUNCTIONS
#################################################################

Function Add-ReportRecord {
    param (
        $Site,
        $Web,
        $AccessType,
        $AccountType,
        $UserEmail,
        $Remarks
        )

    $Record += New-Object PSObject -Property ([ordered]@{
        Title = $Site.Title
        Url = $Site.Url
        IsHubSite = $Site.IsHubSite
        WebTemplate = $Web.WebTemplate
        Created = $Web.Created
        LastItemModifiedDate = $Web.LastItemModifiedDate
        GroupId = $Site.GroupId
        StorageUsageCurrentGB =[Math]::Round(($Site.StorageUsageCurrent/1024),1)
        StorageQuotaGB =[Math]::Round(($Site.StorageQuota/1024),1)
        AccessType = $AccessType
        AccountType = $AccountType
        UserEmail = $UserEmail
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
$FolderPath = "$Env:USERPROFILE\Documents\SPOSolutions\"
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

function Get-ObjectID {
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


function Get-GroupUsers {
    param (
        $LoginName
    )

    Try{
        
        If($LoginName -clike '*_o'){

            $GroupID = Get-ObjectID -LoginName $LoginName
            $GroupUsers = Get-AzureADGroupOwner -ObjectId $GroupID

        }
        Else{

            $GroupID = Get-ObjectID -LoginName $LoginName
            $GroupUsers = Get-AzureADGroupMember -ObjectId $GroupID

        }
    }
    Catch {

       $GroupUsers = ""

    }

    Return $GroupUsers

}

function Get-FullControlUsers {
    param (
        $Site,
        $Web,
        $SiteType
    )

    foreach ( $SiteRoleAssignment in $Web.RoleAssignments ) {

        Get-PnPProperty -ClientObject $SiteRoleAssignment -Property RoleDefinitionBindings, Member

        $SitePermissionLevels = $SiteRoleAssignment.RoleDefinitionBindings | Where-Object { ($_.Name -eq "Full Control") }
        If($SitePermissionLevels.Length -eq 0 -or $SiteRoleAssignment.Member.Title -clike '*Limited Access System Group*') {Continue}

        $SitePermissionType = $SiteRoleAssignment.Member.PrincipalType

        If($SitePermissionType -eq "SharePointGroup") {
            
            $AccessType = "SharePoint Group '$($SiteRoleAssignment.Member.Title)'"

            $GroupMembers = Get-PnPGroupMember -Identity $SiteRoleAssignment.Member.Title
                
            ForEach($GroupMember in $GroupMembers) {
                    
                If($GroupMember.PrincipalType -eq "SecurityGroup"){
                        
                    $AccountType = "Security Group '$($GroupMember.Title)'"

                    $GroupUsers = Get-GroupUsers -LoginName $GroupMember.LoginName
                
                    ForEach($GroupUser in $GroupUsers) {

                        Add-ReportRecord -Site $Site -Web $Web -AccessType $AccessType -AccountType $AccountType -UserEmail $GroupUser.UserPrincipalName

                    }
                }
                Else{
                        
                    if ($GroupMember.LoginName -eq "SHAREPOINT\system") { continue }
                    Add-ReportRecord -Site $Site -Web $Web -AccessType $AccessType -AccountType "User" -UserEmail $GroupMember.LoginName

                }
            }
        }
        else {

            Add-ReportRecord -Site $Site -Web $Web -AccessType "Direct Permission" -AccountType "User" -UserEmail $SiteRoleAssignment.Member.Title

        }
    }
}



try {

    Connect-PnPOnline -Url $AdminSiteURL -Interactive -ErrorAction Stop
    Add-ScriptLog -Color Cyan -Msg "Connected to SharePoint Admin Center"

    #Connect-AzureAD
    Add-ScriptLog -Color Cyan -Msg "Connected to Azure AD"

    $collSiteCollections = Get-PnPTenantSite | Where-Object { ($_.Title -notlike "" -and $_.Template -notlike "*Redirect*") }
    Add-ScriptLog -Color Cyan -Msg "Collected all Site Collections"

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
        
        Set-PnPTenantSite -Url $oSite.Url -Owners $SiteCollAdmin -ErrorAction Stop

        Connect-PnPOnline -Url $oSite.Url -Interactive -ErrorAction Stop
        
        $oWeb = Get-PnPWeb -Includes RoleAssignments, Created, LastItemModifiedDate, WebTemplate -ErrorAction Stop

        Get-FullControlUsers -Site $oSite -Web $oWeb -ErrorAction Stop

        $collSubsites = Get-PnPSubWeb -Recurse -Includes HasUniqueRoleAssignments
        
        ForEach($oSubsite in $collSubsites) {

            $PercentComplete = [math]::Round( $PercentComplete + ( ($ItemCounterStep / ($collSubsites.Count + 1)) * 100 ), 2 )
            Add-ScriptLog -Color Yellow -Msg "$($PercentComplete)% Completed - Processing Subsite: $($oSubsite.Title)"

            if($oSubsite.HasUniqueRoleAssignments){

                try {

                    Connect-PnPOnline -Url $oSubsite.Url -Interactive -ErrorAction Stop
                    $oSubWeb = Get-PnPWeb -Includes RoleAssignments, Created, LastItemModifiedDate, WebTemplate
                    Get-FullControlUsers -Site $oSubsite -Web $oSubWeb  -ErrorAction Stop
            
                }
                catch {

                    Add-ScriptLog -Color Red -Msg "Error while processing Subsite '$($oSubsite.Url)'"
                    Add-ScriptLog -Color Red -Msg "Error message: '$($_.Exception.Message)'"
                    Add-ScriptLog -Color Red -Msg "Error trace: '$($_.Exception.ScriptStackTrace)'"
                    Add-ReportRecord -Site $oSubsite -Remarks $_.Exception.Message
            
                }
            }
            else {

                Add-ReportRecord -Site $oSubsite -Web $oSubWeb -AccessType "Same as Parent Site" -AccountType "Same as Parent Site" -UserEmail "Same as Parent Site"
                Add-ScriptLog -Color DarkYellow -Msg "Same permissions as Parent Site"
                
            }
            
        }
    }
    Catch {
        Add-ScriptLog -Color Red -Msg "Error while processing Site Collection '$($Site.Url)'"
        Add-ScriptLog -Color Red -Msg "Error message: '$($_.Exception.Message)'"
        Add-ScriptLog -Color Red -Msg "Error trace: '$($_.Exception.ScriptStackTrace)'"
        Add-ReportRecord -Site $oSite -Remarks $_.Exception.Message
    }

    Connect-PnPOnline -Url $oSite.Url -Interactive
    Remove-PnPSiteCollectionAdmin -Owners $SiteCollAdmin

}

if($collSiteCollections.Count -ne 0) { 
    $PercentComplete = [math]::Round($ItemCounter/$collSiteCollections.Count * 100, 1) 
    Add-ScriptLog -Color Cyan -Msg "$($PercentComplete)% Completed - Finished running script"
}
Add-ScriptLog -Color Cyan -Msg "Report generated at at $($ReportOutput)"
```

<br>

## Users with access at the Site level

```powershell
#################################################################
# DEFINE PARAMETERS FOR THE CASE
#################################################################
$SPOAdminURL = "https://<DOMAIN>-admin.sharepoint.com"
$AdminUPN = "<ADMIN@EMAIL.com>"



#################################################################
# REPORT AND LOGS FUNCTIONS
#################################################################

Function Add-ReportRecord($Web, $RoleAssignment, $User, $PermissionLevels) {
    

    $Record = New-Object PSObject -Property ([ordered]@{
        "SiteName" = $Web.Title
        "SiteUrl" = $Web.Url -Replace ('.*sharepoint.com','')
        "GroupName" = $RoleAssignment.Member.Title
        "UserName" = $User.Title
        "UserUPN" = $User.UserPrincipalName
        "UserEmail" = $User.Email
        "PermissionLevels" = $PermissionLevels
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
$ReportName = "OwnersMembersReport"
$FolderName = $Date + "_" + $ReportName
New-Item -Path $FolderPath -Name $FolderName -ItemType "directory"

# Files
$ReportOutput = $FolderPath + $FolderName + "\" + $FolderName + "_report.csv"
$LogsOutput = $FolderPath + $FolderName + "\" + $FolderName + "_Logs.txt"

Add-ScriptLog -Color Cyan -Msg "Report will be generated at $($ReportOutput)"



#################################################################
# SCRIPT LOGIC
#################################################################

function Get-SitePermissions {
    param (
        $Site
    )

    Connect-PnPOnline -Url $Site.Url -Interactive -ErrorAction Stop
    
    $Web = Get-PnPWeb -Includes RoleAssignments

    Get-UserPermissions -Web $Web
    
}

function Get-UserPermissions {
    param (
        $Web
    )

    ForEach ($RoleAssignment in $Web.RoleAssignments) {
        
        If ( $RoleAssignment.Member.PrincipalType -eq "SharePointGroup" -and ( $RoleAssignment.Member.Title -notlike "*owner*" -or $RoleAssignment.Member.Title -notlike "*member*" ) ) { continue }
        
        Get-PnPProperty -ClientObject $RoleAssignment -Property RoleDefinitionBindings, Member
        
        $PermissionLevels = ($RoleAssignment.RoleDefinitionBindings | Select-Object -ExpandProperty Name | Where-Object { ($_ -ne "Limited Access") -and ($_ -ne "Web-Only Limited Access")} ) -join ","
        If($PermissionLevels.Length -eq 0) {Continue}

        $collUsers = Get-PnPProperty -ClientObject ($RoleAssignment.Member) -Property Users -ErrorAction SilentlyContinue
        If($collUsers.Count -eq 0) { continue }

        ForEach ($oUser in $collUsers) {
            If($oUser.Title -eq "System Account") {Continue}
            Add-ReportRecord -Web $Web -RoleAssignment $RoleAssignment -User $oUser -PermissionLevels $PermissionLevels
        }
    }
}

try {
    Connect-PnPOnline -Url $SPOAdminURL -Interactive -ErrorAction Stop
    Add-ScriptLog -Color Cyan -Msg "Connected to SharePoint Admin Center"

    $collSiteCollections = Get-PnPTenantSite -ErrorAction Stop | Where-Object{ ($_.Title -notlike "" -and $_.Template -notlike "*Redirect*") }
    Add-ScriptLog -Color Cyan -Msg "Collected Site Collections"
    Add-ScriptLog -Color Cyan -Msg "Number of SiteCollections: $($collSiteCollections.count)"
}
catch {
    Add-ScriptLog -Color Red -Msg "Error: $($_.Exception.Message)"
    break
}

$ItemCounter = 0
$ItemCounterStep = 1 / $collSiteCollections.Count
foreach($oSiteCollection in $collSiteCollections){
    
    $PercentComplete = [math]::Round($ItemCounter/$collSiteCollections.Count * 100, 2)
    Add-ScriptLog -Color Yellow -Msg "$($PercentComplete)% Completed - Processing Site Collection: $($oSiteCollection.Title)"
    $ItemCounter++
    
    try{
        Set-PnPTenantSite -Url $oSiteCollection.Url -Owners $AdminUPN

        Get-SitePermissions -Site $oSiteCollection

        try {
            $collSubsites = Get-PnPSubWeb -Recurse -Includes HasUniqueRoleAssignments

            ForEach($oSubsite in $collSubsites) {

                $PercentComplete = [math]::Round( $PercentComplete + ( ($ItemCounterStep / $collSubsites.Count) * 100 ), 2 )
                Add-ScriptLog -Color Yellow -Msg "$($PercentComplete)% Completed - Processing Subsite: $($oSubsite.Title)"
            
                Get-SitePermissions -Site $oSubsite
            }
        }
        catch {
            Add-ScriptLog -Color Red -Msg "Error processing Subsite: $($oSiteCollection.Url)"
            Add-ScriptLog -Color Red -Msg "Error message: $($_.Exception.Message)"
        }

    }
    catch{
        Add-ScriptLog -Color Red -Msg "Error processing Site Collection: $($oSiteCollection.Url)"
        Add-ScriptLog -Color Red -Msg "Error message: $($_.Exception.Message)"
    }
}
# Close status notification
$PercentComplete = [math]::Round($ItemCounter/$collSiteCollections.Count * 100, 1)
Add-ScriptLog -Color Cyan -Msg "$($PercentComplete)% Completed - Finished running script"
Add-ScriptLog -Color Cyan -Msg "Report generated at at $($ReportOutput)"


```

<br>

## Collect SharePoint Groups, Users and Security Groups

```powershell
#Define Parameters
$SiteURL= "https://<Domain>.sharepoint.com/sites/<SiteName>"
$ReportOutput = "C:\Temp\SitePermissionsReport.csv"

#Connect to PnP Online
Connect-PnPOnline -Url $SiteURL -UseWebLogin

$Global:Results = @()


#Function to add Unique Permissions to the Report
Function Add-Report($ItemObject, $ItemType){
    if($UserPrincipalName -clike "*#ext#*"){
        $UserType = "External"
        }
    else{
        $UserType = "Internal"
        }
    $Global:Results += New-Object PSObject -Property ([ordered]@{
        ItemName               = $ItemName           
        ItemRelativeURL        = $ItemRelativeURL
        ItemType               = $ItemType
        AccessType             = $PermissionType
        GroupName              = $GroupName
        UserName               = $UserName
        UserPrincipalName      = $UserPrincipalName
        UserEmail              = $UserEmail
        UserType               = $UserType
        PermissionLevels       = $PermissionLevels
        })
    }


#Function to Users Unique Permissions and itinerate
Function Get-UserPermissions($RoleAssignments, $Msg){
    ForEach ($RoleAssignment in $RoleAssignments)
        {
        #Get the Permission Levels assigned and Member
        Get-PnPProperty -ClientObject $RoleAssignment -Property RoleDefinitionBindings, Member
     
        #Transform the Permission Levels into string
        $PermissionLevels = ($RoleAssignment.RoleDefinitionBindings | Select -ExpandProperty Name | Where { ($_ -ne "Limited Access") -and ($_ -ne "Web-Only Limited Access")} ) -join ","
        If($PermissionLevels.Length -eq 0) {Continue}

        Write-host -f Yellow "$Msg for"$RoleAssignment.Member.Title
        Write-host -f DarkYellow "at"$ItemRelativeURL

        $PermissionType = $RoleAssignment.Member.PrincipalType
        $GroupName = $RoleAssignment.Member.Title

        #Get Site GROUP Permissions
        If($PermissionType -eq "SharePointGroup")
            {
            If ($RoleAssignment.Member.Title -like "SharingLinks*"){
                $PermissionType = "Shared Link"
                }
            $Users = Get-PnPProperty -ClientObject ($RoleAssignment.Member) -Property Users -ErrorAction SilentlyContinue
            If($Users.Count -eq 0) {
                $UserName               = "None"
                $UserPrincipalName      = "None"
                $UserEmail              = "None"
                Add-Report
                Continue
                }
            Else {
                ForEach ($User in $Users)
                    {
                    If($User.Title -eq "System Account") {Continue}
                    #Report information
                    $UserName               = $User.Title
                    $UserPrincipalName      = $User.UserPrincipalName
                    $UserEmail              = $User.Email
                    Add-Report
                    }
                }
            }
        #Get Site USER Permissions
        Else
            {
            $UserName               = $RoleAssignment.Member.Title
            $UserPrincipalName      = $RoleAssignment.Member.LoginName.Replace('i:0#.f|membership|','')
            $UserEmail              = $RoleAssignment.Member.LoginName.Replace('i:0#.f|membership|','')

            Add-Report
            }
        }
    }


#Get SITE Unique Permissions
$WebRoles = Get-PnPWeb -Includes RoleAssignments

#Report information for Site
$ItemName               = $WebRoles.Title
$ItemRelativeURL        = $WebRoles.Url -Replace ('.*sharepoint.com','')
$ItemType               = "Site"

#Parameters for Get-UserPermissions Command
$SiteMsg = "Getting Site Permissions"
Get-UserPermissions -RoleAssignments $WebRoles.RoleAssignments -Msg $SiteMsg

Write-Host -f Green "Getting Unique Permissions at Site level for"$WebRoles.Title"COMPLETED!" 


#Get all Lists and itinerate
$Lists = Get-PnPList
ForEach ($List in $Lists)
{
    $ListMsg = "Getting Unique Permissions on {0} '{1}'" -f $List.BaseType,$List.Title
    
    #Check if the LIST has Unique Permissions
    $ListHasUniquePermissions = Get-PnPProperty -ClientObject $List -Property "HasUniqueRoleAssignments"
    If($ListHasUniquePermissions)
        {
        #Report information for List
        $ItemName               = $List.Title
        $ItemRelativeURL        = $List.DefaultViewUrl -Replace ('Forms/AllItems.aspx','') -Replace ('AllItems.aspx','')
        $ItemType               = $List.BaseType
        
        #Parameters for Get-UserPermissions Command
        $ListRoles = Get-PNPList -Identity $List -Includes RoleAssignments
        Get-UserPermissions -RoleAssignments $ListRoles.RoleAssignments -Msg $ListMsg
        }


    $ItemCounter = 0

    #Get all Items and itinerate
    $ItemList = Get-PnPListItem -List $List.Title -PageSize 2000
    ForEach($Item in $ItemList)
        {
        #Status notification
        $ItemCounter++
        $ItemProcess = [math]::Round($ItemCounter/$ItemList.Count*100,1)
        Write-Progress -PercentComplete ($ItemCounter / ($ItemList.Count) * 100) -Activity "Processing $($ItemProcess)%" -Status $ListMsg
        
        $ItemMsgAdd = " {0} '{1}'" -f $Item.FileSystemObjectType,$Item.FieldValues["FileLeafRef"]
        $ItemMsg = $ListMsg + $ItemMsgAdd

        #Check if the ITEM has Unique Permissions
        $ItemHasUniquePermissions = Get-PnPProperty -ClientObject $Item -Property "HasUniqueRoleAssignments"
        If($ItemHasUniquePermissions)
            {       
            #Report information for Item
            $ItemName               = $Item.FieldValues["FileLeafRef"]
            $ItemRelativeURL        = $Item.FieldValues["FileRef"]
            $ItemType               = $Item.FileSystemObjectType
            
            #Parameters for Get-UserPermissions Command
            $ItemRoleAssignments = Get-PnPProperty -ClientObject $Item -Property RoleAssignments
            Get-UserPermissions -RoleAssignments $ItemRoleAssignments -Msg $ItemMsg
            }
        }
    Write-Progress -Activity "Processing $($ItemProcess)%" -Status $ListMsg -Completed
    Write-host -f Cyan $ListMsg "COMPLETED"
    }
Write-Host -f Green "Getting Unique Permissions on Document Libraries and Item Lists COMPLETED!"

#Export Report
If (Test-Path $ReportOutput) { Remove-Item $ReportOutput }
$Global:Results | Export-CSV $ReportOutput -NoTypeInformation
Write-host -b Green "Permissions Report Generated Successfully!"
```

<br>

## Collect SharePoint Groups, Users, Security Groups and users part of the security groups

```powershell
#Define Parameters
$SiteURL = "https://DOMAIN.sharepoint.com/sites/SITENAME"
$ReportOutput = "$Env:USERPROFILE\Desktop\SitePermissionsReport.csv"
$Global:Results = @()

#Get Credentials to connect
$Cred  = Get-Credential

#Connect to Services
Connect-PnPOnline -Url $SiteURL –Credential $Cred
Connect-AzureAD –Credential $Cred


# Function to add Unique Permissions to the Report
Function Add-Report(){

    if($Global:UserType -like "*#ext#*"){$Global:UserType = "External"}
    else{$Global:UserType = "Internal"}

    $Global:Results += New-Object PSObject -Property ([ordered]@{
        "Item.Name"                  = $Global:ItemName          
        "Item.RelativeURL"           = $Global:ItemRelativeURL
        "Item.Type"                  = $Global:ItemType
        "Access.Type"                = $Global:PermissionType
        "Sharepoint.Group.Name"      = $Global:SharepointGroupName
        "SecurityGroup.Name"         = $Global:SecurityGroupName
        "SecurityGroup.Email"        = $Global:SecurityGroupEmail
        "User.Name"                  = $Global:UserName
        "User.Email"                 = $Global:UserEmail
        "User.Type"                  = $Global:UserType
        "Permissions.Levels"         = $Global:PermissionLevels
    })
}



# Clean up LoginName to get Group ID
Function Get-AA.ObjectID($LoginName){
    $GroupID = $LoginName
    $GroupID = $GroupID -replace ('_o', '')
    $GroupID = $GroupID -replace ('c:0o.c|federateddirectoryclaimprovider|', '')
    $GroupID = $GroupID -replace ('c:0t.c|tenant|', '')
    $GroupID = $GroupID.Trim('|')

    Return $GroupID
}



# Function to get Security Group Users depending if Added group is owners or only members
Function Get-AA.GroupUsers($LoginName){
    # Get Group Owners
    Try{
        If($LoginName -clike '*_o'){
            $GroupID = Get-AA.ObjectID -LoginName $LoginName
            $GroupUsers = Get-AzureADGroupOwner -ObjectId $GroupID
        }
        # Get Group Members
        Else{
            $GroupID = Get-AA.ObjectID -LoginName $LoginName
            $GroupUsers = Get-AzureADGroupMember  -ObjectId $GroupID
        }
    }
    Catch{
        Clear-Variable -Name "GroupUsers"
    }
    Return $GroupUsers 
}



# Function to analyze the type of member; User or Security Group, and get group users if apply.
Function Check-AA.Member($Member){
    
    If(($Member.Title -eq "System Account") -or ($Member.Title -eq "Everyone") -or ($Member.Title -eq "Everyone except external users")){Continue}
    
    If($Member.PrincipalType -eq "SecurityGroup"){
        
        $GroupUsers = Get-AA.GroupUsers -LoginName $Member.LoginName
        ForEach($User in $GroupUsers){

            $Global:test1 = $Member
            $Global:SecurityGroupName    = $Member.Title
            $Global:SecurityGroupEmail   = $Member.Email
            $Global:UserName             = $User.DisplayName
            $Global:UserEmail            = $User.Mail
            $Global:UserType             = $User.UserPrincipalName
            If($Global:SecurityGroupEmail.Length -eq 0){$Global:SecurityGroupEmail = $Member.LoginName}
            Add-Report
        }
    }
    Else{

        Write-Host "Found user"$Member.Email$Member.UserPrincipalName

        $Global:SecurityGroupName    = ""
        $Global:SecurityGroupEmail   = ""
        $Global:UserName             = $Member.Title
        $Global:UserEmail            = $Member.Email
        $Global:UserType             = $Member.UserPrincipalName
        Add-Report
    }
}



# Function to Users Unique Permissions and itinerate
Function Get-AA.UserPermissions($RoleAssignments, $Msg){
    ForEach ($RoleAssignment in $RoleAssignments){
        $Global:SharepointGroupName  = ""
        $Global:SecurityGroupName    = ""
        $Global:SecurityGroupEmail   = ""
        $Global:UserName             = ""
        $Global:UserEmail            = ""

        # Get the Permission Levels assigned and Member
        Get-PnPProperty -ClientObject $RoleAssignment -Property RoleDefinitionBindings, Member
     
        # Transform the Permission Levels into string
        $Global:PermissionLevels = ($RoleAssignment.RoleDefinitionBindings | Select -ExpandProperty Name | Where { ($_ -ne "Limited Access") -and ($_ -ne "Web-Only Limited Access")} ) -join ","
        If($Global:PermissionLevels.Length -eq 0 -or $SiteRoleAssignment.Member.Title -clike '*Limited Access System Group*') {Continue}
        
        Write-host -f Yellow "$Msg for"$RoleAssignment.Member.Title
        Write-host -f DarkYellow "at"$Global:ItemRelativeURL

        $Global:PermissionType = $RoleAssignment.Member.PrincipalType
        $GroupName = $RoleAssignment.Member.Title

        #Get Site GROUP Permissions
        If($Global:PermissionType -eq "SharePointGroup"){

            $Global:SharepointGroupName   = $RoleAssignment.Member.Title
            
            If ($RoleAssignment.Member.Title -like "SharingLinks*"){$Global:PermissionType = "SharedLink"}

            $GroupMembers = Get-PnPGroupMember -Identity $RoleAssignment.Member.Title
            
            If($GroupMembers.Count -ne 0){ForEach ($Member in $GroupMembers){Check-AA.Member -Member $Member}}
        }

        #Get Site USER Permissions
        Else{Check-AA.Member -Member $RoleAssignment.Member}
    }
}



# Get Site Unique Permissions
$WebRoles = Get-PnPWeb -Includes RoleAssignments

# State information of the current checking item for the report
$Global:ItemName              = $WebRoles.Title
$Global:ItemRelativeURL       = $WebRoles.Url -Replace ('.*sharepoint.com','')
$Global:ItemType              = "Site"

Write-host -f Cyan "Checking Site level"

# Parameters for Get-AA.UserPermissions Command
$SiteMsg = "Getting Site Permissions"
Get-AA.UserPermissions -RoleAssignments $WebRoles.RoleAssignments -Msg $SiteMsg

$ExcludedLists = @("appdata", "appfiles", "Composed Looks", "Converted Forms", "Form Templates", "List Template Gallery", "Master Page Gallery", "Preservation Hold Library", "Project Policy Item List", "Site Assets", "Site Pages", "Solution Gallery", "Style Library", "TaxonomyHiddenList", "Theme Gallery", "User Information List", "Web Part Gallery")

# Get all Lists and iterate
$Lists = Get-PnPList | Where-Object {($_.Hidden -eq $False) -and ($_.Title -notin $ExcludedLists)}
ForEach ($List in $Lists)
{
    Write-host -f Cyan "Checking"$List.BaseType$List.Title

    
    $ListMsg = "Getting Unique Permissions on {0} '{1}'" -f $List.BaseType,$List.Title
    
    #Check if the LIST has Unique Permissions
    $ListHasUniquePermissions = Get-PnPProperty -ClientObject $List -Property "HasUniqueRoleAssignments"
    If($ListHasUniquePermissions)
        {
        #Report information for List
        $Global:ItemName              = $List.Title
        $Global:ItemRelativeURL       = $List.DefaultViewUrl -Replace ('Forms/AllItems.aspx','') -Replace ('AllItems.aspx','')
        $Global:ItemType              = $List.BaseType
        
        #Parameters for Get-AA.UserPermissions Command
        $ListRoles = Get-PNPList -Identity $List -Includes RoleAssignments
        Get-AA.UserPermissions -RoleAssignments $ListRoles.RoleAssignments -Msg $ListMsg
        }
    
    $ItemCounter = 0

    #Get all Items and itinerate
    $ItemList = Get-PnPListItem -List $List.Title -PageSize 2000
    ForEach($Item in $ItemList)
        {
        #Status notification
        $ItemCounter++
        $ItemProcess = [math]::Round($ItemCounter/$ItemList.Count*100,1)
        Write-Progress -PercentComplete ($ItemCounter / ($ItemList.Count) * 100) -Activity "Processing $($ItemProcess)%" -Status $ListMsg
        
        $ItemMsgAdd = " {0} '{1}'" -f $Item.FileSystemObjectType,$Item.FieldValues["FileLeafRef"]
        $ItemMsg = $ListMsg + $ItemMsgAdd

        #Check if the ITEM has Unique Permissions
        $ItemHasUniquePermissions = Get-PnPProperty -ClientObject $Item -Property "HasUniqueRoleAssignments"
        If($ItemHasUniquePermissions)
            {       
            #Report information for Item
            $Global:ItemName              = $Item.FieldValues["FileLeafRef"]
            $Global:ItemRelativeURL        = $Item.FieldValues["FileRef"]
            $Global:ItemType              = $Item.FileSystemObjectType
            
            #Parameters for Get-AA.UserPermissions Command
            $ItemRoleAssignments = Get-PnPProperty -ClientObject $Item -Property RoleAssignments
            Get-AA.UserPermissions -RoleAssignments $ItemRoleAssignments -Msg $ItemMsg
            }
        }
    Write-Progress -Activity "Processing $($ItemProcess)%" -Status $ListMsg -Completed

    }
Write-Host -f Green "Getting Unique Permissions on Document Libraries and Item Lists COMPLETED!"

# Export the results to CSV
If($Global:Results.count -eq 0){
    Write-host -b Red "Report is empty!"
}
Else{
    If (Test-Path $ReportOutput) { Remove-Item $ReportOutput }
    $Global:Results | Export-Csv -Path $ReportOutput -NoTypeInformation
    Write-host -b Green "Report Generated Successfully!"
    Write-host -f Green $ReportOutput
}
```