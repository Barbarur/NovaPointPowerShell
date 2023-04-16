#Report #SharePointOnline #PowerShell #PnP #SiteCollection #SiteClassification #SiteAdmin 

<br>

```powershell
# Define Parameters
$AdminSiteURL= "https://DOMAIN-admin.sharepoint.com"
$ReportOutput = "C:\Temp\SiteClassificationReport.csv"
$Global:Results = @()
$ItemCounter = 0 

# Connect to SharePoint Online Admin Center
Connect-PnPOnline -Url $AdminSiteURL -Interactive

# Add records to the Report
Function Add-Report($AccessType, $GroupName, $AccountType, $AccountName, $SitePermissionLevels){
    $Global:Results += New-Object PSObject -Property ([ordered]@{
        SiteName               = $Site.Title
        SiteURL                = $Site.Url
        SiteClasification      = $SiteClassification
    })
}

#Get all Sites and iterate
$Sites = Get-PnPTenantSite | Where{ ($_.Title -notlike "" -and $_.Template -notlike "*Redirect*") }
Write-Host -f Cyan "Total number of Sites:"$Sites.Count
ForEach($Site in $Sites){
    # Status notification
    $ItemCounter++
    $ItemProcess = [math]::Round($ItemCounter/$Sites.Count*100,1)
    Write-Progress -PercentComplete $ItemProcess -Activity "Processing $($ItemProcess)%" -Status "Site '$($Site.Url)"
    Write-Host -f Yellow $Site.url -NoNewline

    # Connect and get Site
    Connect-PnPOnline -Url $Site.Url -Interactive
    $Site = Get-PnPSite

    Try{
        # Get Site Clasification property
        $SiteClassification = Get-PnPProperty -ClientObject $Site -Property Classification
        Write-Host -f Green ' COMPLETED!'
        Add-Report
    
    }
    Catch{
        $SiteClassification = 'ERROR!'
        Write-Host -f Red ' ERROR!'
        Add-Report
        Write-Host -f Red $_.Exception.Message
    }
}
# Close status notification
Write-Progress -Activity "Processing $($ItemProcess)%" -Status "Site '$($Site.URL)"

# Export the results to CSV
If($Global:Results.count -eq 0){
    Write-host -b Red "Report is empty!"
}
Else{
    If (Test-Path $ReportOutput) { Remove-Item $ReportOutput }
    $Global:Results | Export-Csv -Path $ReportOutput -NoTypeInformation
    Write-host -b Green "Report Generated Successfully!"
    Write-host -f Green $ReportOutput
}
```