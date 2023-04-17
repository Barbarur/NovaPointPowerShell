#Automation #OneDrive #PowerShell #PnP #SPOService  #Storage

<br>

## Option 1: Using SharePoint Management Shell; Set same as organization default limit

```powershell
#Define Parameters
$AdminSiteURL="https://{DOMAIN}-admin.sharepoint.com"

#Get Credentials
$Cred = Get-Credential

#Connect to SharePoint Online Admin Center
Connect-SPOService -Url $AdminSiteURL –credential $Cred

#Get all OneDrive for Business Site collections
$AllSites = Get-SPOSite -Template “SPSPERS” -Limit ALL -IncludePersonalSite $True

Write-Host -f Yellow "Total Number of Sites Found: "$AllSites.count

#Add Site Collection Admin to each OneDrive
Foreach($Site in $AllSites)
{
    Write-Host -f Yellow "Changing OneDrive Quota to: "$Site.URL
    Set-SPOSite -Identity $Site.Url -StorageQuotaReset
}
Write-Host "Site Collection Admin Added to All Sites Successfully!" -f Green
```

<br>

## Option 2: Using SharePoint Management Shell; Set Maximum storage per OneDrive

```powershell
#Define Parameters
$AdminSiteURL="https://{DOMAIN}-admin.sharepoint.com"
$StorageQuota = "5242880"

#Get Credentials
$Cred = Get-Credential

#Connect to SharePoint Online Admin Center
Connect-SPOService -Url $AdminSiteURL –credential $Cred

#Get all OneDrive for Business Site collections
$AllSites = Get-SPOSite -Template “SPSPERS” -Limit ALL -IncludePersonalSite $True

Write-Host -f Yellow "Total Number of Sites Found: "$AllSites.count

#Add Site Collection Admin to each OneDrive
Foreach($Site in $AllSites)
{
    Write-Host -f Yellow "Changing OneDrive Quota to: "$Site.URL
    Set-SPOSite -Identity $Site.Url -StorageQuota $StorageQuota
}
Write-Host "Site Collection Admin Added to All Sites Successfully!" -f Green
```

<br>

## Option #3: Using PnP; Set custom OneDrive storage quota for a list of users using PnP

```powershell
#Define Parameters
$AdminSiteURL="https://{DOMAIN}-admin.sharepoint.com"
$CSVFile = "C:\Temp\UserList.csv"
$StorageQuota = 153600
$StorageQuotaWarning = $StorageQuota * 0.9

#Connect to PnP Online
Connect-PnPOnline $AdminSiteURL -UseWebLogin

#Import User List
$CSVData = Import-CSV $CSVFile
Write-host -f Yellow "Total Number of Users in the List:"$CsVData.Count

ForEach ($Row in $CSVData){
    $UserAccount = $Row.'UPN'
    Set-PnPUserOneDriveQuota -Account $UserAccount -Quota $StorageQuota -QuotaWarning $StorageQuotaWarning
}
```