#Automation #PnP #PowerShell #SharePointOnline #SiteCollection #Subsite #RecycleBin 

<br>

## Clear recycle bin for all Site Collections and Subsites in the tenant

```powershell
#################################################################
# DEFINE PARAMETERS FOR THE CASE
#################################################################
$AdminSiteURL= "https://<DOMAIN>-admin.sharepoint.com"
$StartDate = get-date("10/10/2022")
$EndDate = get-date("10/21/2022")
$DeletedBy = "USER@EMAIL.com"



#################################################################
# REPORT AND LOGS FUNCTIONS
#################################################################
# Add Log of the Script
Function Add-ScriptLog($Color, $Msg)
{
    $Date = Get-Date -Format "yyyy/MM/dd HH:mm K"
    $Msg = $Date + " - " + $Msg
    Add-Content -Path $LogsOutput -Value $Msg
    Write-host -f $Color $Msg
}

# Create logs location
$ReportName = "ClearAllSitesRecycleBin"
$Date = Get-Date -Format FileDateTime
$FolderName = $Date + "_" + $ReportName
$FolderPath = "$Env:USERPROFILE\Documents\"
New-Item -Path $FolderPath -Name $FolderName -ItemType "directory"

# Create logs file
$LogsName = $ReportName + "_Logs.txt"
$LogsOutput = $FolderPath + $FolderName + "\" + $LogsName

Add-ScriptLog -Color Cyan -Msg "Logs will be generated at $($LogsOutput)"



#################################################################
# SCRIPT LOGIC
#################################################################
Function Get-DeletedItems ($ErrorItems){
    return Get-PnPRecycleBinItem -RowLimit 3000 | Where-Object { $_.DeletedDate -gt $StartDate -and $_.DeletedDate -lt $EndDate -and $_.DeletedByEmail -eq $DeletedBy -and $_.id -notin $ErrorItems}
}

function Clear-RecycleBin {

    $DeletedBatch = 1
    $ErrorItems = @()
    $DeletedItems = Get-DeletedItems -ErrorItems $ErrorItems
    while ($DeletedItems.count -ne 0) {
        Add-ScriptLog -Color Yellow -Msg "$($PercentComplete)% Completed - Clearing Recycle bin of site '$($Site.URL)' - Batch $($DeletedBatch)"
        ForEach ($Item in $DeletedItems) {
            try {
                Clear-PnPRecycleBinItem -Identity $DeletedItems.ID
            }
            catch {
                Add-ScriptLog -Color Red -Msg "Error: $($_.Exception.Message)"
                $ErrorItems += $Item.id
            }
        }
        $DeletedBatch++
        $DeletedItems = Get-DeletedItems -ErrorItems $ErrorItems
    }
}

try {
    Connect-PnPOnline -Url $AdminSiteURL -Interactive -ErrorAction Stop
    $SitesList = Get-PnPTenantSite -ErrorAction Stop | Where-Object{ ($_.Title -notlike "" -and $_.Template -notlike "*Redirect*") }
    Add-ScriptLog -Color Cyan -Msg "Connected to SPO Admin Center and Collected Sites"
}
catch {
    Add-ScriptLog -Color Red -Msg "Error: $($_.Exception.Message)"
}

# Iterate through All Sites
$ItemCounter = 0
ForEach($Site in $SitesList)
{
    # Adding notification and logs
    $PercentComplete = [math]::Round($ItemCounter/$SitesList.Count * 100, 0)
    Add-ScriptLog -Color Yellow -Msg "$($PercentComplete)% Completed - Clearing Recycle bin of site '$($Site.URL)'"
    
    try {
        Connect-PnPOnline -Url $Site.Url -Interactive -ErrorAction Stop
    }
    catch {
        Add-ScriptLog -Color Red -Msg "Error connecting to site '$($Site.URL)'"
        Add-ScriptLog -Color Red -Msg "Error: $($_.Exception.Message)"
        Continue
    }

    Clear-RecycleBin
    
    # Get all subsites and iterate
    $SubSites = Get-PnPSubWeb -Recurse
    foreach ($Site in $SubSites) {

        Add-ScriptLog -Color Yellow -Msg "$($PercentComplete)% Completed - Clearing Recycle bin of site '$($Site.URL)'"
        
        try {
            Connect-PnPOnline -Url $Site.Url -Interactive -ErrorAction Stop
        }
        catch {
            Add-ScriptLog -Color Red -Msg "Error connecting to site '$($Site.URL)'"
            Add-ScriptLog -Color Red -Msg "Error: $($_.Exception.Message)"
            Continue
        }

        Clear-RecycleBin
    }

    $ItemCounter++
}
# Close status notification
$PercentComplete = [math]::Round($ItemCounter/$ItemsList.Count * 100, 1)
Add-ScriptLog -Color Cyan -Msg "$($PercentComplete)% Completed - Script finished"
Add-ScriptLog -Color Cyan -Msg "Logs generated at $($LogsOutput)"
```

<br>

## Clear recycle bin of a single Site

```powershell
#################################################################
# DEFINE PARAMETERS FOR THE CASE
#################################################################
$SiteURL= "https://<DOMAIN>.sharepoint.com/sites/<SITENAME>"
$StartDate = get-date("10/10/2022")
$EndDate = get-date("10/21/2022")
$DeletedBy = "USER@EMAIL.com"



#################################################################
# REPORT AND LOGS FUNCTIONS
#################################################################
# Add Log of the Script
Function Add-ScriptLog($Color, $Msg)
{
    $Date = Get-Date -Format "yyyy/MM/dd HH:mm K"
    $Msg = $Date + " - " + $Msg
    Add-Content -Path $LogsOutput -Value $Msg
    Write-host -f $Color $Msg
}

# Create logs location
$ReportName = "ClearSiteRecycleBin"
$Date = Get-Date -Format FileDateTime
$FolderName = $Date + "_" + $ReportName
$FolderPath = "$Env:USERPROFILE\Documents\"
New-Item -Path $FolderPath -Name $FolderName -ItemType "directory"

# Create logs file
$LogsName = $ReportName + "_Logs.txt"
$LogsOutput = $FolderPath + $FolderName + "\" + $LogsName

Add-ScriptLog -Color Cyan -Msg "Logs will be generated at $($LogsOutput)"



#################################################################
# SCRIPT LOGIC
#################################################################
Function Get-DeletedItems ($ErrorItems){
    return Get-PnPRecycleBinItem -RowLimit 3000 | Where-Object { $_.DeletedDate -gt $StartDate -and $_.DeletedDate -lt $EndDate -and $_.DeletedByEmail -eq $DeletedBy -and $_.id -notin $ErrorItems}
}

function Clear-RecycleBin {

    $DeletedBatch = 1
    $ErrorItems = @()
    $DeletedItems = Get-DeletedItems -ErrorItems $ErrorItems
    while ($DeletedItems.count -ne 0) {
        Add-ScriptLog -Color Yellow -Msg "$($PercentComplete)% Completed - Clearing Recycle bin of site '$($Site.URL)' - Batch $($DeletedBatch)"
        ForEach ($Item in $DeletedItems) {
            try {
                Clear-PnPRecycleBinItem -Identity $DeletedItems.ID
            }
            catch {
                Add-ScriptLog -Color Red -Msg "Error: $($_.Exception.Message)"
                $ErrorItems += $Item.id
            }
        }
        $DeletedBatch++
        $DeletedItems = Get-DeletedItems -ErrorItems $ErrorItems
    }
}

try {
    Connect-PnPOnline -Url $SiteURL -Interactive -ErrorAction Stop
    Add-ScriptLog -Color Cyan -Msg "Connected to Site '$($SiteURL)'"
}
catch {
    Add-ScriptLog -Color Red -Msg "Error: $($_.Exception.Message)"
}

Clear-RecycleBin

# Close status notification
Add-ScriptLog -Color Cyan -Msg "Script finished"
Add-ScriptLog -Color Cyan -Msg "Logs generated at $($LogsOutput)"
```