#Automation #AccessControl #ExternalSharing #OneDrive #SharePointOnline #Permissions #PowerShell #PnP #SPOService #SiteCollection 

<br>

## Option 1: Using PnP; Remove External users from a single Site Collection

```powershell
#Define Parameters
$SiteURL= "https://DOMAIN.sharepoint.com/sites/SITENAME"
   
#Connect to PnP Online
Connect-PnPOnline -Url $SiteURL -UseWebLogin
$Web = Get-PnPWeb

$UserCounter = 0 
#Get All users who have permission to the subsite
$Users = Get-PnPUser -WithRightsAssigned -Web $Web | Where {$_.LoginName -clike "*#ext#*"}
ForEach ($User in $Users){
    
    # Status notification
    $UserCounter++
    $UserProcess = [math]::Round($UserCounter/$Users.Count*100,1)
    Write-Progress -PercentComplete $UserProcess -Activity "Processing $($UserProcess)%" -Status "Removing $($User.Email)"
    
    Remove-PnPUser -Identity $User.LoginName -Confirm:$false
    Write-Host -f Yellow "Removed User:"$User.Email
    
    }

# Close status notification
Write-Progress -Activity "Processing $($UserProcess)%" -Status "Removing $($User.Email)"

Write-Host -b Green "Finished creating folders!"
```

<br>
## Option 2: Using SharePoint Management Shell; Remove External users from all Site Collections

```powershell
# Define Parameters
$AdminSiteURL = "https://DOMAIN-admin.sharepoint.com"
$ReportOutput = "C:\RemovedUsersReport.csv"
$Global:Results = @()
$ItemCounter = 0

# Connect to SharePoint Online Admin Center
Connect-SPOService -Url $AdminSiteURL

#Add records to the Report
Function Add-Report($SiteName, $SiteURL, $UserDisplayName, $UserLoginName, $Action){
    $Global:Results += New-Object PSObject -Property ([ordered]@{
        SiteName               = $SiteName
        SiteURL                = $SiteURL
        UserDisplayName        = $UserDisplayName
        UserLoginName          = $UserLoginName
        Action                 = $Action
    })
}


# Get all Site collections
$Sites = Get-SPOSite -Limit ALL -IncludePersonalSite $False
Write-Host 'Total number of Site Collections:'$Sites.Count

# Itinerate across all Sites
ForEach($Site in $Sites){
    $ItemCounter++
    $ItemProcess = [math]::Round($ItemCounter/$Sites.Count*100,1)
    Write-Progress -PercentComplete $ItemProcess -Activity "Processing $($ItemProcess)%" -Status "Site '$($Site.URL)"
    
    Write-Host -f Yellow $Site.url


    # COLLECT AND REMOVE EXTERNAL USERS
    Try{
        $EXTUsers = Get-SPOUser -Site $Site.Url -Limit ALL | Where{$_.UserType -eq "Guest"}
        
        ForEach($EXTUser in $EXTUsers){
            Remove-SPOUser -Site $Site.Url -LoginName $EXTUser.LoginName
            Write-Host -f Green $Site.Url$EXTUser.LoginName "REMOVED"

            Add-Report -SiteName $Site.Title -SiteURL $Site.Url -UserDisplayName $EXTUser.DisplayName -UserLoginName $EXTUser.LoginName -Action 'Removed from Site'
        }
    }
    Catch{
        Write-Host -f Red $Site.url"ERROR while collecting and removing external users!"
    }

}
# Close status notification
Write-Progress -Activity "Processing $($ItemProcess)%" -Status "Site '$($Site.URL)"



If($Global:Results.count -eq 0){
    Write-host -b Green "Report is empty!"
}
Else{
    #Export the results to CSV
    If (Test-Path $ReportOutput) { Remove-Item $ReportOutput }
    $Global:Results | Export-Csv -Path $ReportOutput -NoTypeInformation
    Write-Host -b Green "Finished removing users!"
    Write-host -f Green $ReportOutput
}
```