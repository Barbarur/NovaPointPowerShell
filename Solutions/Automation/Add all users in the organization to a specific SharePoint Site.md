#Automation #PowerShell #Pnp #Msol #Permissions #SharePointOnline 

<br>

```powershell
# Define Parameters
$SiteURL = "https://DOMAIN.sharepoint.com/sites/SITENAME"
$GroupName ="SP GROUP NAME"
 
# Connect to Services
Connect-PnPOnline -Url $SiteURL
Connect-MsolService

# Get all users and iterate
$Users = Get-MsolUser -All
$ItemCounter = 0 
ForEach($User in $Users){
    # Status notification
    $ItemCounter++
    $ItemProcess = [math]::Round($ItemCounter/$Users.Count*100,1)
    Write-Progress -PercentComplete $ItemProcess -Activity "Processing $($ItemProcess)%" -Status "User: '$($User.UserPrincipalName)"

    # Add current user to the SharePoint Group
    Add-PnPGroupMember -LoginName $User.UserPrincipalName -Identity $GroupName
}
# Close status notification
Write-Progress -Activity "Processing $($ItemProcess)%" -Status "User: '$($User.UserPrincipalName)"

Write-Host -b Green "Finished adding new users to the Site!"
```