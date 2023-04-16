#Automation #PowerShell #PnP #CSV #ExternalSharing #SharedLink #SharePointOnline #SiteCollection 

<br>

## Set External Sharing restriction by Allowed domains at <u>Tenant level</u>

```powershell
#Define Parameters
$TenantAdminURL = "https://{DOMAIN}-admin.sharepoint.com"

#Define Allowed Domains List.
$ListDomains = "XXX.com XXX.com"

#Connect to SPO Admin Center
Connect-SPOService -Url $TenantAdminURL -Credential (Get-Credential)
 
#Enable Policy and Space delimited list of allowed Domains
Set-SPOTenant -SharingDomainRestrictionMode "AllowList" -SharingAllowedDomainList $ListDomains
```

<br>

## Set External Sharing restriction by Domain at <u>Site level</u>

```powershell
#Define Parameters
$TenantAdminURL = "https://{DOMAIN}-admin.sharepoint.com"
$SiteURL = "https://{DOMAIN}.sharepoint.com/sites/{SITENAME}"
 
#Define Allowed Domains List.
$ListDomains = "XXX.com XXX.com"

#Get Credentials
$Cred  = Get-Credential

#Connect to Admin Center
Connect-SPOService -Url $TenantAdminURL -Credential $Cred
 
#Space delimited list of allowed Domains
Set-SPOSite -Identity $SiteURL -SharingDomainRestrictionMode "AllowList" -SharingAllowedDomainList $ListDomains
```

<br>

## Set External Sharing restriction by Domain for all <u>OneDrive Sites</u>

```powershell
#Define Parameters
$TenantAdminURL = "https://{DOMAIN}-admin.sharepoint.com"
 
#Define Allowed Domains List.
$ListDomains = "XXX.com XXX.com"

#Get Credentials
$Cred  = Get-Credential

#Connect to Admin Center
Connect-SPOService -Url $TenantAdminURL -Credential $Cred

#Get all OneDrive for Business Site collections
$AllSites = Get-SPOSite -Template “SPSPERS” -Limit ALL -IncludePersonalSite $True

#Loop through each site
ForEach($Site in $AllSites)
{ 
    Write-Host -f Yellow "Processing Site: "$Site.URL

    Set-SPOSite -Identity $Site.URL -SharingDomainRestrictionMode "AllowList" -SharingAllowedDomainList $ListDomains 
}
```

<br>

## Set External Sharing restriction by Allowed domains assigning specific domains per site

Create a CSV with the URL of each Site and the allowed domains at _C:\Temp\SiteDomains.csv_. Make sure each domain is separated by a blank space.

| URL | DOMAINS |
| :--- | :--- |
| https://DOMAIN.sharepoint.com/sites/SITENAME1 | AAA.com BBB.com CCC.com |
| https://DOMAIN.sharepoint.com/sites/SITENAME2 | DDD.com EEE.com FFF.com |

```powershell
#Define Parameters
$TenantAdminURL = "https://DOMAIN-admin.sharepoint.com"
$CSVFile = "C:\Temp\SiteDomains.csv"

#Get Credentials
$Cred  = Get-Credential

#Connect to Admin Center
Connect-SPOService -Url $TenantAdminURL -Credential $Cred

# Import Sites and Domains
$CSVData = Import-CSV $CSVFile

# Iterate through the sites
$ItemCounter = 0
ForEach($Row in $CSVData){
    $ItemCounter++
    $ItemProcess = [math]::Round($ItemCounter/$CSVData.Count*100,1)
    Write-Progress -PercentComplete $ItemProcess -Activity "Processing $($ItemProcess)%" -Status "Site '$($Row.URL)"
    Write-host -f Yellow $Row.Url -NoNewline
    
    Try{
        Set-SPOSite -Identity $Row.Url -SharingDomainRestrictionMode "AllowList" -SharingAllowedDomainList $Row.Domains
        Write-host -f Green " COMPLETED"
        }
    Catch{
        Write-Host -f Red " ERROR!"
        }
}
# Close status notification
Write-Progress -Activity "Processing $($ItemProcess)%" -Status "Site '$($Row.URL)"

Write-Host -b Green "Finished setup domain policies!"
```

<br>

## Set External Sharing restriction by Allowed domains for all sites excluding a Whitelist of sites

Create a CSV file with the URL of the Sites that will be on the White List at _C:\Temp\WhileList.csv_ . You only need the column ‘URL’, the others are for you to easily track the each Site.

| Name | UPN | URL |
| :--- | :--- | :--- |
| Alex Wilber | alex.wilber@domain.com | https://domain-my.sharepoint.com/personal/alex_wilber_domain_com
| Joni Sherman | joni.sherman@domain.com | https://domain-my.sharepoint.com/personal/joni_sherman_domain_com
| Megan Bowen | megan.bowen@domain.com | https://domain-my.sharepoint.com/personal/megan_bowen_domain_com

```powershell
#Define Parameters
$AdminSiteURL = "https://DOMAIN-admin.sharepoint.com"
$CSVFile = "C:\Temp\WhileList.csv"

#Create a list of domains
$GrossListDomain = @'
gmail.com
hotmail.com
'@
$ListDomains = (-split $GrossListDomain) -join " "

#Get Credentials to connect
$Cred  = Get-Credential

#Connect to SharePoint Online Admin Center
Connect-SPOService -Url $AdminSiteURL –Credential $Cred

#Import User List
$CSVData = Import-CSV $CSVFile

#Get all OneDrive for Business Site collections
$AllSites = Get-SPOSite -Limit ALL -IncludePersonalSite $True
Set-SPOTenant -SharingDomainRestrictionMode "None"

$ItemCounter = 0 
#Loop through each site
ForEach($Site in $AllSites)
{ 
    $ItemCounter++
    $ItemProcess = [math]::Round($ItemCounter/$AllSites.Count*100,1)
    Write-Progress -PercentComplete ($ItemCounter / ($AllSites.Count) * 100) -Activity "Processing $($ItemProcess)%" -Status "Site '$($Site.URL)"
    If($CSVData.URL -contains $Site.URL)
    {
    Set-SPOSite -Identity $Site.URL -SharingDomainRestrictionMode "None"
    Write-Host -f Green "Excluded Site:"$Site.URL
    }
    else
    {
    Set-SPOSite -Identity $Site.URL -SharingDomainRestrictionMode "AllowList" -SharingAllowedDomainList $ListDomains
    Write-Host -f Yellow "Limit Sharing by domain policy applied to"$Site.URL
    }
}
```

<br>

## Potential modifications to the script

### Set External Sharing restriction by Blocked domains

In case that instead of allowing only External sharing to specific domains, you want to restrict the external sharing with specific domains, you can change the command _Set_SPOSite_ for the one below.

```powershell
Set-SPOSite -Identity $Site.URL -SharingDomainRestrictionMode "BlockList" -SharingBlockedDomainList $ListDomains
```

### Add an new domain to the list

Replace the section _Define Allowed Domains List_ of the scripts above by the one showed here below.

```powershell
# Get list of domains that are allowed to share.
$ListDomains = Get-SPOTenant | Select -ExpandProperty SharingAllowedDomainList
 
# Add a new domain to the list.
$ListDomains = $ListDomains.Insert(0,"XXX.com ")
```

### Remove a domain from the list

Replace the section _Define Allowed Domains List_ of the scripts above by the one showed here below .
```powershell
# Get list of domains that are allowed to share.
$ListDomains = Get-SPOTenant | Select -ExpandProperty SharingAllowedDomainList
 
# Remove a domain from the list.
$ListDomains = $ListDomains.Replace("XXX.com","")
```

### Long Domain List

When managing a long list of allowed domains, getting them concatenated with a space in-between as needed by the cmdlet, can be a troublesome. In order to manage this list easier, you can keep the list on a Word or Excel document that you keep updated as needed.

Then you can replace the section _Define Allowed Domains List_ of the scripts above by the one showed here below, copy the list from your document and paste it directly on the script. It will create automatically the concatenated list needed for the script to run smoothly.

```powershell
# Create a list of domains
$GrossListDomain = @'
XXX
XXX
'@
$ListDomains = (-split $GrossListDomain) -join " "
```