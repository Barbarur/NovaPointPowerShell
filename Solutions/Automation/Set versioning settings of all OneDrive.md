
<br>

## Change Major versions for Documents Library all Site Collections

```powershell
#Define Parameters
$AdminSiteURL="https://DOMAIN-admin.sharepoint.com"
$VersionsLimit = 10
 
#Connect to PnP Online
Connect-PnPOnline -Url $AdminSiteURL -UseWebLogin
  
#Get All Sites
$AllSites = Get-PnPTenantSite -IncludeOneDriveSites -Filter "Url -like '-my.sharepoint.com/personal/'"

$ItemCounter = 0 
ForEach($Site in $AllSites)
{
    #Connect to OneDrive for Business Site
    Connect-PnPOnline $Site.URL -Credentials $Cred
    
    $ItemCounter++
    Write-Host -f Yellow "Processing Site:"$Site.URL
    Write-Progress -PercentComplete ($ItemCounter / ($AllSites.Count) * 100) -Activity "Processing Files $ItemCounter of $($AllSites.Count)" -Status "Processing Site '$($Site.URL)"
        
    Set-PnPList -Identity "Documents" -EnableVersioning $true -MajorVersions $VersionsLimit
    }
```

<br>

## Change Major and Minor versions for all Libraries and List in all Site Collections and Subsites

```powershell
#Define Parameters
$AdminSiteURL="https://DOMAIN-admin.sharepoint.com"
$SiteCollAdmin = "EMAIL@EMAIL.com"
$MajorLimit = 100
$MinorLimit = 100



# Set versioning limitation
Function Set-VersionToAllLists($SiteURL){
    Connect-PnPOnline -Url $SiteURL -Interactive

    $ExcludedLists = @("appdata", "appfiles", "Composed Looks", "Converted Forms", "Form Templates", "List Template Gallery", "Master Page Gallery", "Preservation Hold Library", "Project Policy Item List", "Site Assets", "Site Pages", "Solution Gallery", "Style Library", "TaxonomyHiddenList", "Theme Gallery", "User Information List", "Web Part Gallery")
    $Lists = Get-PnPList | Where-Object {($_.Hidden -eq $False) -and ($_.Title -notin $ExcludedLists)}
    ForEach ($List in $Lists){
        # Botification of current List
        Write-Host -f Cyan "Modifying versioning for $($List.BaseType) '$($List.Title)'"
        Write-Host -f DarkCyan $List.DefaultViewUrl
        
        # Set versioning limitation
        #If($List.BaseType -eq 'DocumentLibrary'){Set-PnPList -Identity $List.Title -EnableVersioning $true -EnableMinorVersions $true -MajorVersions $MajorLimit -MinorVersions $MinorLimit}
        #If($List.BaseType -eq 'GenericList'){Set-PnPList -Identity $List.Title -EnableVersioning $true -MajorVersions $MajorLimit}
    }
}



#Connect to PnP Online
Connect-PnPOnline -Url $AdminSiteURL -Interactive
  
#Get all Site Collections and iterate
$Sites = Get-PnPTenantSite | Where{ ($_.Title -notlike "" -and $_.Template -notlike "*Redirect*") }
$ItemCounter = 0 
ForEach($Site in $Sites)
{
    # Add user as Site Collection Admin
    Set-PnPTenantSite -Url $Site.Url -Owners $SiteCollAdmin

    # Status notification
    $ItemCounter++
    $ItemProcess = [math]::Round($ItemCounter/$Sites.Count*100,1)
    Write-Progress -PercentComplete $ItemProcess -Activity "Processing $($ItemProcess)%" -Status "Site '$($Site.URL)"
    Write-host -f Yellow "Modifying versioning for Site Collection: $($Site.URL)"

    # Set version limit to current Site Collection
    Set-VersionToAllLists -SiteURL $Site.URL

    # Get all subsites
    $SubSites = Get-PnPSubWeb
    ForEach($SubSite in $SubSites){
        # Set version limit to current Subsite
        Set-VersionToAllLists -SiteURL $SubSite.url
    }

    # Remove user as Site Collection Admin
    Remove-PnPSiteCollectionAdmin -Owners $SiteCollAdmin
}
# Close status notification
Write-Progress -Activity "Processing $($ItemProcess)%" -Status "Site '$($Site.URL)"
Write-Host -b Green "Finished changing versioning settings!"
```