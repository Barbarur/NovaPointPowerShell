#Automation #AuditLog #ExchangeOnline #PowerShell #SharePointOnline 

<br>

## Audit Log for all Activities in a specific URL

You need to have the exchange module installed. In case you don’t have it, here you have the commands.
```powershell
Install-Module -Name ExchangeOnlineManagement
Import-Module ExchangeOnlineManagement
```

Connect to Exchange first

```powershell
$Cred = Get-Credential
Connect-ExchangeOnline –Credential $Cred
```

Run the script for Audit Log script. Potential changes can be found below the script.

```powershell
#Define Parameters
$logFile = "C:\Temp\AuditLogSearchLog.txt"
$outputFile = "C:\Temp\AuditLogRecords.csv"
[DateTime]$start = [DateTime]::UtcNow.AddDays(-7)
[DateTime]$end = [DateTime]::UtcNow
$ObjectIDs = "https://{DOMAIN}.sharepoint.com/sites/{SITENAME}/Shared Documents/*"
$resultSize = 5000
$intervalMinutes = 60

#Start script
[DateTime]$currentStart = $start
[DateTime]$currentEnd = $start

Function Write-LogFile ([String]$Message)
{
    $final = [DateTime]::Now.ToUniversalTime().ToString("s") + ":" + $Message
    $final | Out-File $logFile -Append
}

Write-LogFile "BEGIN: Retrieving audit records between $($start) and $($end), RecordType=$record, PageSize=$resultSize."
Write-Host "Retrieving audit records for the date range between $($start) and $($end), RecordType=$record, ResultsSize=$resultSize"

$totalCount = 0
while ($true)
{
    $currentEnd = $currentStart.AddMinutes($intervalMinutes)
    if ($currentEnd -gt $end)
    {
        $currentEnd = $end
    }

    if ($currentStart -eq $currentEnd)
    {
        break
    }

    $sessionID = [Guid]::NewGuid().ToString() + "_" +  "ExtractLogs" + (Get-Date).ToString("yyyyMMddHHmmssfff")
    Write-LogFile "INFO: Retrieving audit records for activities performed between $($currentStart) and $($currentEnd)"
    Write-Host "Retrieving audit records for activities performed between $($currentStart) and $($currentEnd)"
    $currentCount = 0

    $sw = [Diagnostics.StopWatch]::StartNew()
    do
    {
        $results = Search-UnifiedAuditLog -StartDate $currentStart -EndDate $currentEnd -ObjectIDs $ObjectIDs -SessionId $sessionID -SessionCommand ReturnLargeSet -ResultSize $resultSize


        if (($results | Measure-Object).Count -ne 0)
        {
            $results | export-csv -Path $outputFile -Append -NoTypeInformation

            $currentTotal = $results[0].ResultCount
            $totalCount += $results.Count
            $currentCount += $results.Count
            Write-LogFile "INFO: Retrieved $($currentCount) audit records out of the total $($currentTotal)"

            if ($currentTotal -eq $results[$results.Count - 1].ResultIndex)
            {
                $message = "INFO: Successfully retrieved $($currentTotal) audit records for the current time range. Moving on!"
                Write-LogFile $message
                Write-Host "Successfully retrieved $($currentTotal) audit records for the current time range. Moving on to the next interval." -foregroundColor Yellow
                ""
                break
            }
        }
    }
    while (($results | Measure-Object).Count -ne 0)

    $currentStart = $currentEnd
}

Write-LogFile "END: Retrieving audit records between $($start) and $($end), RecordType=$record, PageSize=$resultSize, total count: $totalCount."
Write-Host "Script complete! Finished retrieving audit records for the date range between $($start) and $($end). Total count: $totalCount" -foregroundColor Green
```

<br>

## Audit Log only for specific activities

Add values on the top of the script. Full list of activities that can be search [here](https://docs.microsoft.com/en-us/microsoft-365/compliance/search-the-audit-log-in-security-and-compliance?view=o365-worldwide#file-and-page-activities).

```powershell
$AuditOperations = @('FileAccessed','FileAccessedExtended','FileDownloaded','FolderDeleted')
```

Change row 45

```powershell
$results = Search-UnifiedAuditLog -StartDate $currentStart -EndDate $currentEnd -ObjectIDs $ObjectIDs -SessionId $sessionID -SessionCommand ReturnLargeSet -ResultSize $resultSize -Operations $AuditOperations
```

<br>

#### Reference

- [Use a PowerShell script to search the audit log](https://docs.microsoft.com/en-us/microsoft-365/compliance/audit-log-search-script?view=o365-worldwide)
- [Connect to Exchange Online PowerShell](https://docs.microsoft.com/en-us/powershell/exchange/connect-to-exchange-online-powershell?view=exchange-ps)
- [Search-UnifiedAuditLog](https://docs.microsoft.com/en-us/powershell/module/exchange/search-unifiedauditlog?view=exchange-ps)
- [File and page activities](https://docs.microsoft.com/en-us/microsoft-365/compliance/search-the-audit-log-in-security-and-compliance?view=o365-worldwide#file-and-page-activities)