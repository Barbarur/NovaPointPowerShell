#Automation #ExternalUser #ExternalSharing #OneDrive #SharePointOnline #Permissions #PowerShell #PnP #SiteCollection 

<br>

## Remove all External users from all Site Collections

```powershell
# Define Parameters
$AdminSiteURL = "https://<Domain>-admin.sharepoint.com"
$SiteCollAdmin = "Admin@email.com"
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
$Sites = Get-SPOSite -Limit ALL -IncludePersonalSite $False | Where {$_.url -eq 'https://DOMAIN.sharepoint.com/sites/SITENAME' -or $_.url -eq 'https://DOMAIN.sharepoint.com/sites/SITENAME'}
Write-Host 'Total number of Site Collections:'$Sites.Count

# Itinerate across all Sites
ForEach($Site in $Sites){
    $ItemCounter++
    $ItemProcess = [math]::Round($ItemCounter/$Sites.Count*100,1)
    Write-Progress -PercentComplete $ItemProcess -Activity "Processing $($ItemProcess)%" -Status "Site '$($Site.URL)"
    
    Write-Host -f Yellow $Site.url
    
    
    # ADD USER AS ADMIN
    Try{
        Set-SPOUser -Site $Site.Url -LoginName $SiteCollAdmin -IsSiteCollectionAdmin $True
        Add-Report -SiteName $Site.Title -SiteURL $Site.Url -UserDisplayName $SiteCollAdmin -UserLoginName $SiteCollAdmin -Action 'Added as Admin'
    }
    Catch{
        Write-Host -f Red $Site.url"ERROR Adding user as Admin!"
        Add-Report -SiteName $Site.Title -SiteURL $Site.Url -UserDisplayName $SiteCollAdmin -UserLoginName $SiteCollAdmin -Action 'ERROR Adding user as Admin!'
    }


    # COLLECT AND REMOVE EXTERNAL USERS
    Try{
        $EXTUsers = Get-SPOUser -Site $Site.Url -Limit ALL | Where{$_.UserType -eq "Guest"}
        
        ForEach($EXTUser in $EXTUsers){
            #Remove-SPOUser -Site $Site.Url -LoginName $EXTUser.LoginName
            Write-Host -f Green $Site.Url$EXTUser.LoginName "REMOVED"

            Add-Report -SiteName $Site.Title -SiteURL $Site.Url -UserDisplayName $EXTUser.DisplayName -UserLoginName $EXTUser.LoginName -Action 'Removed from Site'
        }
    }
    Catch{
        Write-Host -f Red $Site.url"ERROR while collecting and removing external users!"
        Add-Report -SiteName $Site.Title -SiteURL $Site.Url -UserDisplayName $SiteCollAdmin -UserLoginName $SiteCollAdmin -Action 'ERROR while collecting and removing external users!'
    }

    
    # REMOVE USER AS ADMIN
    Try{
        Set-SPOUser -Site $Site.Url -LoginName $SiteCollAdmin -IsSiteCollectionAdmin $False
        Add-Report -SiteName $Site.Title -SiteURL $Site.Url -UserDisplayName $SiteCollAdmin -UserLoginName $SiteCollAdmin -Action 'Removed as Admin'
    }
    Catch{
        Write-Host -f Red $Site.url"ERROR Removing user as Admin!"
        Add-Report -SiteName $Site.Title -SiteURL $Site.Url -UserDisplayName $SiteCollAdmin -UserLoginName $SiteCollAdmin -Action 'ERROR Removing user as Admin!'
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