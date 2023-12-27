#Automation #SharePointOnline #PowerShell #PnP #DocumentLibrary #Subsite #ItemList 

<br>

Subsites cannot be set as Read-Only by using [Site Policies or Locking the site](https://novacato.com/how-to-make-a-site-view-only-without-changing-the-permission-levels/), these solutions apply to the full Site Collection. In the scenario of Subsite, we will need to change all permissions to ‘Read’.

This can be a big manual work, which can be automatized by using PowerShell.

<br>

## Option 1: Delete all Unique Permissions for Library/List and Folders/Files/Items

This script will:

-   Stop inheriting permissions for the Subsite from the Site Collection.
-   Set permissions of all SharePoint Groups and User with Direct access to the Site as “Restricted View”.
-   Restore Permissions Inheritance for all List and Libraries.
-   Restore Permission Inheritance for all Folders, Files and Items.

```powershell
#Define Parameters
$SiteURL= "https://DOMAIN.sharepoint.com/sites/SITENAME/SUBSITENAME/SUBSUBSITENAME"
$PermissionToAdd = "Restricted View"

#Connect to PnP Online
Connect-PnPOnline -Url $SiteURL -UseWebLogin


#Stop Inheriting Permissions of the Subsite
$Web = Get-PnPWeb
$Web.BreakRoleInheritance($True, $False)
Invoke-PnPQuery
Write-Host -b Green "Stop inheriting permissions for the Site COMPLETED"


#Change Groups and Users Permissions on Site
$WebRoles = Get-PnPWeb -Includes RoleAssignments
ForEach ($SiteRoleAssignment in $WebRoles.RoleAssignments)
    {
    #Get the Permission Levels assigned and Member
    Get-PnPProperty -ClientObject $SiteRoleAssignment -Property RoleDefinitionBindings, Member
     
    #Get the Permission Levels assigned
    $SitePermissionLevels = $SiteRoleAssignment.RoleDefinitionBindings | Where { ($_.Name -ne "Limited Access") -and ($_.Name -ne $PermissionToAdd)}
    If($SitePermissionLevels.Length -eq 0) {Continue}
    If($SiteRoleAssignment.Member.Title -clike '*Limited Access System Group*') {Continue}
 
    Write-host -f Yellow "Changing Unique Permissions for"$SiteRoleAssignment.Member.Title

    ForEach($SitePermissionLevel in $SitePermissionLevels)
        {
        $SitePermissionType = $SiteRoleAssignment.Member.PrincipalType
        #Change Site Group Permissions
        If($SitePermissionType -eq "SharePointGroup")
            {
            Set-PnPGroup -Identity $SiteRoleAssignment.Member.Title -AddRole $PermissionToAdd -RemoveRole $SitePermissionLevel.Name
            }
        #Change Site User Permissions
        Else
            {
            $SiteUserUPN = $SiteRoleAssignment.Member.LoginName.Replace('i:0#.f|membership|','')
            Set-PnPWebPermission -User $SiteUserUPN -AddRole $PermissionToAdd -RemoveRole $SitePermissionLevel.Name
            }
        }
    }
Write-Host -b Green "Change Groups and Users Permissions for the Site COMPLETED!"


#Restore Permissions on Libraries/List and Folders/Files/Items
$Lists = Get-PnPList
ForEach ($List in $Lists)
    {
    #Restore Permissions Inheritance for the LIST
    $Msg = "Restoring Permissions Inheritance on {0} '{1}'" -f $List.BaseType,$List.Title
    Write-host -f Yellow $Msg
    Set-PnPList -Identity $List.Title -ResetRoleInheritance
    
    #Get all list items and Iterate through each
    $Items = Get-PnPListItem -List $List.Title -PageSize 2000
    ForEach($Item in $Items)
        {
    #Check if the ITEM has Unique Permissions
    $HasUniquePermissions = Get-PnPProperty -ClientObject $Item -Property "HasUniqueRoleAssignments"
        If($HasUniquePermissions)
            {
            $Msg = "Restoring Permissions Inheritance on {0} '{1}' {2} '{3}'" -f $List.BaseType,$List.Title,$Item.FileSystemObjectType,$Item.FieldValues["FileLeafRef"]
            Write-host -f Yellow $Msg
            Write-host -f DarkYellow "at"$Item.FieldValues["FileRef"]

            #Restore Permissions Inheritance for the ITEM
            Set-PnPListItemPermission -List $List.Title -Identity $Item.ID -InheritPermissions
            }
        }
    Write-host -f Cyan "Restoring Permissions Inheritance on"$List.Title"COMPLETED"
    }
Write-Host -b Green "Restoring Permissions Inheritance on Libraries and Lists COMPLETED"
```

<br>

## Option 2: Set all Unique Permissions as Restricted View

-   Stop inheriting permissions for the Subsite from the Site Collection.
-   Set permissions of all SharePoint Groups and User with Direct access to the Site as “Restricted View”.
-   Set permissions of all SharePoint Groups and User with Direct access to the List/Libraries with Unique Permissions as “Restricted View”.
-   Set permissions of all SharePoint Groups and User with Direct access to the Folders/Files/Items with Unique Permissions as “Restricted View”.
```powershell
#Define Parameters
$SiteURL= "https://DOMAIN.sharepoint.com/sites/SITENAME/SUBSITENAME/SUBSUBSITENAME"
$PermissionToAdd = "Restricted View"

#Connect to PnP Online
Connect-PnPOnline -Url $SiteURL -UseWebLogin


#Stop Inheriting Permissions of the subsite
$Web = Get-PnPWeb
$Web.BreakRoleInheritance($True, $False)
Invoke-PnPQuery
Write-Host -b Green "Stop inheriting permissions for the Site COMPLETED!"


#Change Groups and Users Permissions on Site
$WebRoles = Get-PnPWeb -Includes RoleAssignments
ForEach ($SiteRoleAssignment in $WebRoles.RoleAssignments)
    {
    #Get the Permission Levels assigned and Member
    Get-PnPProperty -ClientObject $SiteRoleAssignment -Property RoleDefinitionBindings, Member
     
    #Get the Permission Levels assigned
    $SitePermissionLevels = $SiteRoleAssignment.RoleDefinitionBindings | Where { ($_.Name -ne "Limited Access") -and ($_.Name -ne $PermissionToAdd)}
    If($SitePermissionLevels.Length -eq 0) {Continue}
    If($SiteRoleAssignment.Member.Title -clike '*Limited Access System Group*') {Continue}
    
    Write-host -f Yellow "Changing Site Permissions for"$SiteRoleAssignment.Member.Title
    
    ForEach($SitePermissionLevel in $SitePermissionLevels)
        {
        $SitePermissionType = $SiteRoleAssignment.Member.PrincipalType
        #Change Site Group Permissions
        If($SitePermissionType -eq "SharePointGroup")
            {
            Set-PnPGroup -Identity $SiteRoleAssignment.Member.Title -AddRole $PermissionToAdd -RemoveRole $SitePermissionLevel.Name
            }
        #Change Site User Permissions
        Else
            {
            $SiteUserUPN = $SiteRoleAssignment.Member.LoginName.Replace('i:0#.f|membership|','')
            Set-PnPWebPermission -User $SiteUserUPN -AddRole $PermissionToAdd -RemoveRole $SitePermissionLevel.Name
            }
        }
    }
Write-Host -b Green "Change Groups and Users Permissions for the Site COMPLETED!" 


#Get all Lists and itinerate
$Lists = Get-PnPList
ForEach ($List in $Lists)
    {
    
    $ListMsg = "Getting Unique Permissions on {0} '{1}'" -f $List.BaseType,$List.Title
    
    #Check if the LIST has Unique Permissions
    $ListHasUniquePermissions = Get-PnPProperty -ClientObject $List -Property "HasUniqueRoleAssignments"
    If($ListHasUniquePermissions)
        {
        #Get all users and groups who has access
        $ListRoles = Get-PNPList -Identity $List -Includes RoleAssignments
        Foreach ($ListRoleAssignment in $ListRoles.RoleAssignments)
            {
            #Get the Permission Levels assigned and Member
            Get-PnPProperty -ClientObject $ListRoleAssignment -Property RoleDefinitionBindings, Member

            #Pull Permission Levels names
            $ListPermissionLevels = $ListRoleAssignment.RoleDefinitionBindings | Where { ($_.Name -ne "Limited Access") -and ($_.Name -ne $PermissionToAdd)}
            If($ListPermissionLevels.Length -eq 0) {Continue}
            
            Write-host -f Yellow "$ListMsg for"$ListRoleAssignment.Member.Title
            $MsgLoc = $List.DefaultViewUrl -Replace ('Forms/AllItems.aspx','') -Replace ('AllItems.aspx','')
            Write-host -f DarkYellow "at"$MsgLoc
            
            $ListPermissionType = $ListRoleAssignment.Member.PrincipalType

            ForEach ($ListPermissionLevel in $ListPermissionLevels)
                {
                #Change GROUP Permissions
                If($ListPermissionType -eq "SharePointGroup")
                    {
                    Set-PnPGroupPermissions -Identity $ListRoleAssignment.Member.Title -List $List.Title -AddRole $PermissionToAdd -RemoveRole $ListPermissionLevel.Name
                    }
                #Change USER Permissions
                Else
                    {
                    $ListUserUPN = $ListRoleAssignment.Member.LoginName.Replace('i:0#.f|membership|','')
                    Set-PnPListPermission -Identity $List.Title -User $ListUserUPN -AddRole $PermissionToAdd -RemoveRole $ListPermissionLevel.Name
                    }
                }
            }
        }

    $ItemCounter = 0

    #Get all ITEMS and iterate through each
    $Items = Get-PnPListItem -List $List.Title -PageSize 2000
    ForEach($Item in $Items)
        {
        #Status notification
        $ItemCounter++
        $ItemProcess = [math]::Round($ItemCounter/$Items.Count*100,1)
        Write-Progress -PercentComplete ($ItemCounter / ($Items.Count) * 100) -Activity "Processing $($ItemProcess)%" -Status $ListMsg
        
        #Check if the ITEM has Unique Permissions
        $ItemHasUniquePermissions = Get-PnPProperty -ClientObject $Item -Property "HasUniqueRoleAssignments"
        If($ItemHasUniquePermissions)
            {       
            #Get all users and groups who has access
            $ItemRoleAssignments = Get-PnPProperty -ClientObject $Item -Property RoleAssignments
            Foreach ($ItemRoleAssignment in $ItemRoleAssignments)
                {
                #Get the Permission Levels assigned and Member
                Get-PnPProperty -ClientObject $ItemRoleAssignment -Property RoleDefinitionBindings, Member

                $ItemPermissionLevels = ($ItemRoleAssignment.RoleDefinitionBindings | Where { ($_.Name -ne "Limited Access") -and ($_.Name -ne $PermissionToAdd)} )
                If($ItemPermissionLevels.Length -eq 0) {Continue}
                If($ItemRoleAssignment.Member.Title -clike '*SharingLinks*') {Continue}

                $ItemMsgAdd = " {0} '{1}'" -f $Item.FileSystemObjectType,$Item.FieldValues["FileLeafRef"]
                $ItemMsg = $ListMsg + $ItemMsgAdd
                Write-host -f Yellow "$ItemMsg for"$ItemRoleAssignment.Member.Title
                Write-host -f DarkYellow "at"$Item.FieldValues["FileRef"]

                $ItemPermissionType = $ItemRoleAssignment.Member.PrincipalType
                
                ForEach ($ItemPermissionLevel in $ItemPermissionLevels)
                    {
                    #Change permissions for FOLDERS
                    If($Item.FileSystemObjectType -eq "Folder")
                        {
                        If($ItemPermissionType -eq "SharePointGroup")
                            {
                            Set-PnPFolderPermission -List $List.Title -Identity $Item.FieldValues["FileRef"] -Group $ItemRoleAssignment.Member.Title -AddRole $PermissionToAdd -RemoveRole $ItemPermissionLevel.Name
                            }
                        Else
                            {
                            $FolderUserUPN = $ItemRoleAssignment.Member.LoginName.Replace('i:0#.f|membership|','')
                            Set-PnPFolderPermission -List $List.Title -Identity $Item.FieldValues["FileRef"] -User $FolderUserUPN -AddRole $PermissionToAdd -RemoveRole $ItemPermissionLevel.Name
                            }
                        }
                    #Change permissions for FILES/ITEMS
                    Else
                        {
                        If($ItemPermissionType -eq "SharePointGroup")
                            {
                            Set-PnPListItemPermission -List $List.Title -Identity $Item.ID -Group $ItemRoleAssignment.Member.Title -AddRole $PermissionToAdd -RemoveRole $ItemPermissionLevel.Name
                            }
                        Else
                            {
                            $FileUserUPN = $ItemRoleAssignment.Member.LoginName.Replace('i:0#.f|membership|','')
                            Set-PnPListItemPermission -List $List.Title -Identity $Item.ID -User $FileUserUPN -AddRole $PermissionToAdd -RemoveRole $ItemPermissionLevel.Name
                            }
                        }
                    }
                }
            }
        }
    Write-Progress -Activity "Processing $($ItemProcess)%" -Status $ListMsg -Completed
    Write-host -f Cyan $ListMsg"COMPLETED"
    }
Write-Host -b Green "Changing Unique Permissions on Document Libraries and Item Lists COMPLETED!"
```