#Automation #OneDrive #SharePointOnline #PnP #PHL #Restore

<br>

```powershell
$SiteURL= "https://<Domain>.sharepoint.com/sites/<SiteName>/"
$ListName= "Preservation Hold Library"
$User = "*User@email.com*"
$StartDate = "2022-01-31 12:00:00 AM"
$EndDate = "2022-03-1 12:00:00 AM"
$TargetLocation = "/sites/<SiteName>/<LibraryName>/"
$ReportOutput = "$Env:USERPROFILE\Desktop\RestoredFilesPHLReport.csv"
$Global:Results = @()
$ItemCounter = 0 

# Connect to PnP Online
Connect-PnPOnline -Url $SiteURL -Interactive

# Add new log to report function
Function Add-Report(){
    $Global:Results += New-Object PSObject -Property ([ordered]@{
        "File Name"                = $Item.FieldValues.FileLeafRef
        "File Original Path"       = $Item.FieldValues.PreservationOriginalURL
        "Modified"                 = $Item.FieldValues.Modified
        "Modified by"              = $Item.FieldValues.Modified_x0020_By
        "Date Preserved"           = $Item.FieldValues.PreservationDatePreserved
    })
}


$Items = Get-PnPListItem -List $ListName -PageSize 2000 | Where{$_.FieldValues.Modified_x0020_By -like $User -and $_.FieldValues.PreservationDatePreserved -gt $StartDate -and $_.FieldValues.PreservationDatePreserved -lt $EndDate}
ForEach($Item in $Items){
    # Status notification
    $ItemCounter++
    $ItemProcess = [math]::Round($ItemCounter/$Items.Count*100,1)
    Write-Progress -PercentComplete $ItemProcess -Activity "Processing $($ItemProcess)%" -Status "File '$($Item.FieldValues.FileLeafRef)"
    
    Try{
        $SourceUrl = $Item.FieldValues.FileRef
        $TargetUrl = $TargetLocation + $Item.FieldValues.FileLeafRef
        Copy-PnPFile -SourceUrl $SourceUrl -TargetUrl $TargetUrl -Force -ErrorAction Stop
	Add-Report
    }
    Catch{
    }
    Write-Host -f Yellow $Item.FieldValues.FileLeafRef
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