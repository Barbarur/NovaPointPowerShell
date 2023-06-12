#Report #SiteCollection #PowerShell #PnP #ExternalSharing 

<br>

## 1. Add yourself in all Sites

## 2. Run Script

## 3. Remove yourself from all Sites

<br>

## Scripts

### Get report of all Site Collections with external users

```powershell
# Define Parameters
$AdminSiteURL = "https://DOMAIN-admin.sharepoint.com"
$ReportOutput = "$Env:USERPROFILE\Desktop\SiteWithExternalUsersReport.csv"
$Global:Results = @()
$ItemCounter = 0

# Connect to SharePoint Online Admin Center
Connect-SPOService -Url $AdminSiteURL

# Add records to the Report
Function Add-Report($Action){
    $Global:Results += New-Object PSObject -Property ([ordered]@{
        SiteName               = $Site.Title
        SiteURL                = $Site.url
    })
}


# Get all Site collections and iterate
$Sites = Get-SPOSite -Limit ALL -IncludePersonalSite $False | Where{ ($_.Title -notlike "") -and ($_.Template -notlike "*Redirect*") }
Write-Host -f Cyan 'Total number of Site Collections:'$Sites.Count
ForEach($Site in $Sites){
    
    # Status notification
    $ItemCounter++
    $ItemProcess = [math]::Round($ItemCounter/$Sites.Count*100,1)
    Write-Progress -PercentComplete $ItemProcess -Activity "Processing $($ItemProcess)%" -Status "Site '$($Site.URL)"
    Write-Host -f Yellow $Site.url

    # Check if site contains external users and add to the report
    Try{
        
        $EXTUsers = Get-SPOUser -Site $Site.Url -Limit ALL | Where{$_.UserType -eq "Guest"}
        If($EXTUsers.length -ne 0){
            Add-Report
        }
    }
    
    Catch{
        Write-Host -f Red $Site.url"ERROR while checking external users!"
    }

}

# Close status notification
Write-Progress -Activity "Processing $($ItemProcess)%" -Status "Site '$($Site.URL)"


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

### Get report of all OneDrive with external users
```powershell
# Define Parameters
$AdminSiteURL = "https://DOMAIN-admin.sharepoint.com"
$SiteCollAdmin = "XXX@XXX.com"
$ReportOutput = "$Env:USERPROFILE\Desktop\SiteWithExternalUsersReport.csv"
$Global:Results = @()
$ItemCounter = 0

# Connect to SharePoint Online Admin Center
Connect-SPOService -Url $AdminSiteURL

# Get all Site collections and iterate
$Sites= Get-SPOSite -Template "SPSPERS" -Limit ALL -IncludePersonalSite $True | Where{ ($_.Title -notlike "" -and $_.Template -notlike "*Redirect*") }
Write-Host -f Cyan 'Total number of Site Collections:'$Sites.Count
ForEach($Site in $Sites){
    
    # Status notification
    $ItemCounter++
    $ItemProcess = [math]::Round($ItemCounter/$Sites.Count*100,1)
    Write-Progress -PercentComplete $ItemProcess -Activity "Processing $($ItemProcess)%" -Status "Site '$($Site.URL)"
    Write-Host -f Yellow $Site.url

    # Check if site contains external users and add to the report
    Try{
        
        Set-SPOUser -Site $Site.Url -LoginName $SiteCollAdmin -IsSiteCollectionAdmin $True
        
        $ListEXTUsers = Get-SPOUser -Site $Site.Url -Limit ALL | Where{$_.UserType -eq "Guest"}
    
        ForEach($User in $ListEXTUsers)
        {
        
            $Global:Results += New-Object PSObject -Property ([ordered]@{
                SiteName               = $Site.Title
                SiteURL                = $Site.url
                UserName               = $User.DisplayName
                UserEmail              = $User.LoginName
                Type                   = $User.UserType
            })
        
        }
    
    }
    
    Catch{
        Write-Host -f Red $Site.url"ERROR while checking external users!"
    }

    Set-SPOUser -Site $Site.Url -LoginName $SiteCollAdmin -IsSiteCollectionAdmin $False

}

# Close status notification
Write-Progress -Activity "Processing $($ItemProcess)%" -Status "Site '$($Site.URL)"


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

<br>

#### References
[Get-SPOExternalUser](https://learn.microsoft.com/en-us/powershell/module/sharepoint-online/get-spoexternaluser?view=sharepoint-ps)