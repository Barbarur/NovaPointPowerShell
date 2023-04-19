#Automation #PowerShell #PnP #SharePointOnline #SiteCollection #Permissions #ItemList #DocumentLibrary 

<br>

## Option 1: Delete Unique Permissions for all items in a specific Document Library or Item List.

```powershell
#Define Parameters
$SiteURL= "https://DOMAIN.sharepoint.com/sites/SITENAME/SUBSITENAME/SUBSUBSITENAME"
$ListName = "Documents"

#Connect to PnP Online
Connect-PnPOnline -Url $SiteURL -UseWebLogin

#Process notification parameters
$ItemCounter = 0
$Msg = "Restoring Permissions Inheritance on '{1}'" -f $ListName
Write-host -f Yellow $Msg -NoNewline

#Get all list items and Iterate through each
$Items = Get-PnPListItem -List $ListName -PageSize 2000
ForEach($Item in $Items)
    {
    #Process notification start 
    $ItemCounter++
    $ItemProcess = [math]::Round($ItemCounter/$Items.Count*100,1)
    Write-Progress -PercentComplete ($ItemCounter / ($Items.Count) * 100) -Activity "$Msg $ItemProcess%" -Status ($Item["FileRef"])
        
    #Check if the Item has Unique Permissions
    $HasUniquePermissions = Get-PnPProperty -ClientObject $Item -Property "HasUniqueRoleAssignments"
    If($HasUniquePermissions)
        {
        #Restore Permissions Inheritance for the ITEM
        Set-PnPListItemPermission -List $ListName -Identity $Item.ID -InheritPermissions
        }
    }
#Process notification finish
Write-Progress -Activity "Processing $($ItemCounter)%" -Status $Msg -Completed
Write-Host -b Green "COMPLETED!"
```

<br>

## Option 2: Delete Unique Permissions for all items in a All Document Libraries and Item Lists of a Site Collection.

```powershell
#Define Parameters
$SiteURL= "https://DOMAIN.sharepoint.com/sites/SITENAME/SUBSITENAME/SUBSUBSITENAME"

#Connect to PnP Online
Connect-PnPOnline -Url $SiteURL -UseWebLogin


#Restore Permissions on Libraries/List and Folders/Files/Items
$Lists = Get-PnPList
ForEach ($List in $Lists)
    {
    #Process notification parameters
    $ItemCounter = 0
    $Msg = "Restoring Permissions Inheritance on {0} '{1}'" -f $List.BaseType,$List.Title
    Write-host -f Yellow $Msg -NoNewline
    
    #Restore Permissions Inheritance for the List
    Set-PnPList -Identity $List.Title -ResetRoleInheritance
    
    #Get all list items and Iterate through each
    $Items = Get-PnPListItem -List $List.Title -PageSize 2000
    ForEach($Item in $Items)
        {
        #Process notification start 
        $ItemCounter++
        $ItemProcess = [math]::Round($ItemCounter/$Items.Count*100,1)
        Write-Progress -PercentComplete ($ItemCounter / ($Items.Count) * 100) -Activity "$Msg $ItemProcess%" -Status ($Item["FileRef"])

        #Check if the Item has Unique Permissions
        $HasUniquePermissions = Get-PnPProperty -ClientObject $Item -Property "HasUniqueRoleAssignments"
        If($HasUniquePermissions)
            {
            #Restore Permissions Inheritance for the ITEM
            Set-PnPListItemPermission -List $List.Title -Identity $Item.ID -InheritPermissions
            }
        }
    #Process notification finish
    Write-Host -f Green "COMPLETED!"
    Write-Progress -Activity "Processing $($ItemCounter)%" -Status $Msg -Completed
    }
Write-Host -b Green "Restoring Permissions Inheritance on Libraries and Lists COMPLETED"
```