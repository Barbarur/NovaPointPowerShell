#Report #SharePointOnline #PowerShell #PnP #DocumentLibrary #ItemList #SharedLink #SiteCollection 

<br>

## Preparations

1.  Ensure your **PowerShell** is [ready to work with all the necessary modules installed](https://novacato.com/get-powershell-ready-before-start-to-work/).
2.  Check the [error code list](https://novacato.com/tag/error-code/) in case you meet any.

## Run script

1.  [Get only the links currently in use](https://novacato.com/get-shared-links-permissions-report-of-a-sharepoint-online-site-using-powershell/#active-links).
2.  [Get all the Shared Links, including the ones without people added](https://novacato.com/get-shared-links-permissions-report-of-a-sharepoint-online-site-using-powershell/#all-links).

#### Scripts

## Option 1: Get only the links currently in use

```powershell
# Parameters
$SiteURL= "https://DOMAIN.sharepoint.com/sites/SITENAME"
$ReportOutput = "C:\Temp\SharedLinkPermissions.csv"

#Connect to PnP Online
Connect-PnPOnline -Url $SiteURL -UseWebLogin
$Ctx = Get-PnPContext
$Results = @()

#Get all Lists and itinerate
$ExcludedLists = @("appdata", "appfiles", "Composed Looks", "Converted Forms", "Form Templates", "List Template Gallery", "Master Page Gallery", "Preservation Hold Library", "Project Policy Item List", "Site Assets", "Site Pages", "Solution Gallery", "Style Library", "TaxonomyHiddenList", "Theme Gallery", "User Information List", "Web Part Gallery")
$Lists = Get-PnPList | Where-Object {$_.Hidden -eq $False -and $_.Title -notin $ExcludedLists}
ForEach ($List in $Lists)
    {
    
    $ListMsg = "Getting Shared Links on {0} '{1}'" -f $List.BaseType,$List.Title
    $ItemCounter = 0
    
    #Get all Items and itinerate
    $Items = Get-PnPListItem -List $List.Title -PageSize 2000
    ForEach($Item in $Items)
        {
        #Status notification
        $ItemCounter++
        $ItemProcess = [math]::Round($ItemCounter/$Items.Count*100,1)
        Write-Progress -PercentComplete ($ItemCounter / ($Items.Count) * 100) -Activity "Processing $($ItemProcess)%" -Status $ListMsg


        $ItemMsgAdd = " for {0} '{1}'" -f $Item.FileSystemObjectType,$Item.FieldValues["FileLeafRef"]
        $ItemMsg = $ListMsg + $ItemMsgAdd

        #Check if the ITEM has Unique Permissions
        $HasUniquePermissions = Get-PnPProperty -ClientObject $Item -Property "HasUniqueRoleAssignments"
        If($HasUniquePermissions)
            {
            #Get Assigned permissions and itinerate
            $RoleAssignments = Get-PnPProperty -ClientObject $Item -Property RoleAssignments
            ForEach($RoleAssignment in $RoleAssignments)
                {
                #Get the Permission Levels assigned and Member
                Get-PnPProperty -ClientObject $RoleAssignment -Property RoleDefinitionBindings, Member
                
                #Skip report if not a Shared Link
                If($RoleAssignment.Member.Title -notlike "SharingLinks*") {Continue}
                
                #Get Users
                $Users = Get-PnPProperty -ClientObject ($RoleAssignment.Member) -Property Users -ErrorAction SilentlyContinue
                
                #Skip report if no users in the Shared Link
                If($Users.Count -eq 0) {Continue}
                
                #Write Logs    
                Write-host -f Yellow $ItemMsg $RoleAssignment.Member.Title
                Write-host -f DarkYellow "at"$Item.FieldValues["FileRef"]

                If($RoleAssignment.Member.Description -clike "*Anonymous*") {$LinkType = "Anonymous"}
                Elseif($RoleAssignment.Member.Description -clike "*Flexible*") {$LinkType = "Specific People"}
                Elseif($RoleAssignment.Member.Description -clike "*Organization*") {$LinkType = "People in your organization"}
                Else{$LinkType = "Other"}

                $PermissionLevels = ($RoleAssignment.RoleDefinitionBindings | Select -ExpandProperty Name) -join ","
                ForEach ($User in $Users)
                    {
                    $Results += New-Object PSObject -property $([ordered]@{
                    ItemName               = $Item.FieldValues["FileLeafRef"]
                    ItemRelativeURL        = $Item.FieldValues["FileRef"]
                    ItemType               = $Item.FileSystemObjectType
                    LinkName               = $RoleAssignment.Member.Title
                    LinkType               = $LinkType
                    UserName               = $User.Title
                    UserPrincipalName      = $User.UserPrincipalName
                    UserEmail              = $User.Email
                    PermissionLevels       = $PermissionLevels
                        })
                    }
                }
            }
        }
    Write-Progress -Activity "Processing $($ItemProcess)%" -Status $ListMsg -Completed
    Write-host -f Cyan $ListMsg"COMPLETED"
    }

# Export the results to CSV
If (Test-Path $ReportOutput) { Remove-Item $ReportOutput }
$Results | Export-CSV $ReportOutput -NoTypeInformation
Write-host -b Green "Sharing Links Report Generated Successfully!"
Write-host -f Green $ReportOutput
```

<br>

## Option #2: Get all the Shared Links, including the ones without people added

```powershell
# Parameters
$SiteURL= "https://DOMAIN.sharepoint.com/sites/SITENAME"
$ReportOutput = "C:\Temp\SharedLinkPermissions.csv"

#Connect to PnP Online
Connect-PnPOnline -Url $SiteURL -UseWebLogin
$Ctx = Get-PnPContext
$Global:Results = @()

$StartTime = Get-Date
Write-Host -f Magenta $StartTime

#Function to add Unique Permissions to the Report
Function Add-Report(){
    $Global:Results += New-Object PSObject -Property ([ordered]@{
        ItemName               = $ItemName
        ItemRelativeURL        = $ItemRelativeURL
        ItemType               = $ItemType
        LinkName               = $LinkName
        LinkType               = $LinkType
        UserName               = $UserName
        UserPrincipalName      = $UserPrincipalName
        UserEmail              = $UserEmail
        PermissionLevels       = $PermissionLevels
        })
    }

#Get all Lists and itinerate
$ExcludedLists = @("appdata", "appfiles", "Composed Looks", "Converted Forms", "Form Templates", "List Template Gallery", "Master Page Gallery", "Preservation Hold Library", "Project Policy Item List", "Site Assets", "Site Pages", "Solution Gallery", "Style Library", "TaxonomyHiddenList", "Theme Gallery", "User Information List", "Web Part Gallery")
$Lists = Get-PnPList | Where-Object {$_.Hidden -eq $False -and $_.Title -notin $ExcludedLists}
ForEach ($List in $Lists)
    {
    
    $ListMsg = "Getting Shared Links on {0} '{1}'" -f $List.BaseType,$List.Title
    $ItemCounter = 0
    
    #Get all Items and itinerate
    $Items = Get-PnPListItem -List $List.Title -PageSize 2000
    ForEach($Item in $Items)
        {
        #Status notification
        $ItemCounter++
        $ItemProcess = [math]::Round($ItemCounter/$Items.Count*100,1)
        Write-Progress -PercentComplete ($ItemCounter / ($Items.Count) * 100) -Activity "Processing $($ItemProcess)%" -Status $ListMsg


        $ItemMsgAdd = " for {0} '{1}'" -f $Item.FileSystemObjectType,$Item.FieldValues["FileLeafRef"]
        $ItemMsg = $ListMsg + $ItemMsgAdd

        #Check if the ITEM has Unique Permissions
        $HasUniquePermissions = Get-PnPProperty -ClientObject $Item -Property "HasUniqueRoleAssignments"
        If($HasUniquePermissions)
            {
            #Get Assigned permissions and itinerate
            $RoleAssignments = Get-PnPProperty -ClientObject $Item -Property RoleAssignments
            ForEach($RoleAssignment in $RoleAssignments)
                {
                #Get the Permission Levels assigned and Member
                Get-PnPProperty -ClientObject $RoleAssignment -Property RoleDefinitionBindings, Member
                
                #Skip report if not a Shared Link
                If($RoleAssignment.Member.Title -notlike "SharingLinks*") {Continue}
                
                #Get Users
                $Users = Get-PnPProperty -ClientObject ($RoleAssignment.Member) -Property Users -ErrorAction SilentlyContinue
                
                If($RoleAssignment.Member.Description -clike "*Anonymous*") {$LinkType = "Anonymous"}
                Elseif($RoleAssignment.Member.Description -clike "*Flexible*") {$LinkType = "Specific People"}
                Elseif($RoleAssignment.Member.Description -clike "*Organization*") {$LinkType = "People in your organization"}
                Else{$LinkType = "Other"}

                $ItemName               = $Item.FieldValues["FileLeafRef"]
                $ItemRelativeURL        = $Item.FieldValues["FileRef"]
                $ItemType               = $Item.FileSystemObjectType
                $LinkName               = $RoleAssignment.Member.Title
                $PermissionLevels       = ($RoleAssignment.RoleDefinitionBindings | Select -ExpandProperty Name) -join ","

                #Write Logs    
                Write-host -f Yellow $ItemMsg $RoleAssignment.Member.Title
                Write-host -f DarkYellow "at"$Item.FieldValues["FileRef"]

                #Skip report if no users in the Shared Link
                If($Users.Count -eq 0) {
                    $UserName               = "None"
                    $UserPrincipalName      = "None"
                    $UserEmail              = "None"
                    Add-Report
                    Continue
                    }

                ForEach ($User in $Users)
                    {
                    $UserName               = $User.Title
                    $UserPrincipalName      = $User.UserPrincipalName
                    $UserEmail              = $User.Email
                    Add-Report
                    }
                }
            }
        }
    Write-Progress -Activity "Processing $($ItemProcess)%" -Status $ListMsg -Completed
    Write-host -f Cyan $ListMsg"COMPLETED"
    }

#Export Report
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

$EndTime = Get-Date
Write-Host -f Magenta $EndTime
```