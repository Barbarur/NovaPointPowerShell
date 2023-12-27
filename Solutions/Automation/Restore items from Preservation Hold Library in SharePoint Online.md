#Report #SharePointOnline #OneDrive #PHL #PreservationHolLibrary #DocumentLibrary #SiteCollection 

<br>

## Copy files deleted during a specific period by a specific user

```powershell
# Define Parameters
$SiteURL = "https://<DOMAIN>.sharepoint.com/sites/<SITENAME>"
$ListName= "Preservation Hold Library"
$User = "*<USER@RMAIL.COM>*"
$StartDate = "2022-01-31 12:00:00 AM"
$EndDate = "2022-03-1 12:00:00 AM"
$TargetLocation = "/sites/<SITENAME>/<DOCUMENTLIBRARY>/"
$ReportOutput = "$Env:USERPROFILE\Desktop\RestoredFilesPHLReport.csv"

# Add new log to report function
Function Add-Report($Action){
    $Global:Results += New-Object PSObject -Property ([ordered]@{
        "File Name"                = $Item.FieldValues.FileLeafRef
        "File Original Path"       = $Item.FieldValues.PreservationOriginalURL
        "Modified"                 = $Item.FieldValues.Modified
        "Modified by"              = $Item.FieldValues.Modified_x0020_By
        "Date Preserved"           = $Item.FieldValues.PreservationDatePreserved
        "Action"                   = $Action
    })
}

# Connect to PnP Online
Connect-PnPOnline -Url $SiteURL -Interactive

# Get ItemsList that match conditions and iterate
$ItemsList = Get-PnPListItem -List $ListName -PageSize 2000 | Where{$_.FieldValues.Modified_x0020_By -like $User -and $_.FieldValues.PreservationDatePreserved -gt $StartDate -and $_.FieldValues.PreservationDatePreserved -lt $EndDate}
$Global:Results = @()
$ItemCounter = 0 
ForEach($Item in $ItemsList) {

    # Status notification
    $ItemCounter++
    $PercentComplete = [math]::Round($ItemCounter/$ItemsList.Count * 100, 1)
    Write-Progress -PercentComplete $PercentComplete -Activity "Processing $($PercentComplete)%" -Status "File $($Item.FieldValues.FileLeafRef)"
    
    Try {
        Write-host -f Yellow "Copying item:"$Item.FieldValues.FileRef -NoNewline
        
        $SourceUrl = $Item.FieldValues.FileRef
        $TargetUrl = $TargetLocation + $Item.FieldValues.FileLeafRef
        Copy-PnPFile -SourceUrl $SourceUrl -TargetUrl $TargetUrl -Force -ErrorAction Stop
        
        Add-Report ("Completed")
        Write-host -f Green "COMPLETED"
    }
    Catch {
        Write-host -f Red "ERROR"
        Add-Report ("Error")
        Write-host "Error: $($_.Exception.Message)" -foregroundcolor Red
    }
}

# Close status notification
Write-Progress -Activity "Processing $($PercentComplete)%" -Status "Site '$($Site.URL)"
Write-host -f Green "Restored $($ItemsList.Count) items."

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