#Automation #PowerShell #PnP #CSV

<br>

## Create CSV file with folder structure

| Country | BU | Department
| :--- | :--- | :---
| UK | BU#1 | Finance
| Australia | BU#2 | Legal
| New Zealand | BU#3 | Procurement
| Canada | BU#4 | Logistics
| India | BU#5 | Operation
| | BU#6 | Sales
| | BU#7 | HR
| | BU#8
| | BU#9

<br>

## Run Script

```powershell
# Define Parameters
$SiteURL = "https://DOMAIN.sharepoint.com/sites/SITENAME"
$Libraryname = 'LIBRARYNAME'
$CSVFile = "$Env:USERPROFILE\Desktop\Folders.csv"

# Connect to SharePoint Online
Connect-PnPOnline -Url $SiteURL -Interactive

# Import User List
$CSVData = Import-CSV $CSVFile

# Get Column Headers
$Columns = $CSVData[0].psobject.Properties.name

# Set list of locations to be used on the iterations
$Locations = @($Libraryname)
$NextLocations = @()

# Iterate Through Columns
ForEach($Column in $Columns){

    $ItemCounter = 0

    # Iterate through each location where to create new folders
    ForEach($Location in $Locations){
        
        # Get all items in a column, not all columns are expected to have the same number of items
        $ColumnItems = $CSVData.$Column | Where{$_ -ne ''}

        # Iterate through each Item for the folder to be created
        ForEach($Item in $ColumnItems){
            # Status notification
            $ItemCounter++
            $TotalItems = $ColumnItems.Count * $Locations.count
            $ItemProcess = [math]::Round($ItemCounter/$TotalItems*100,1)
            Write-Progress -PercentComplete $ItemProcess -Activity "Processing $($ItemProcess)%" -Status "Creating Folders for '$($Column)'"

            # Create Folder
            Add-PnPFolder -Name $Item -Folder $Location

            # Add location for next column iteration
            $NewFolder = $Location + '/' + $Item
            $NextLocations += $NewFolder
        }
    }

    # Modify the location list for the next iteration
    $Locations = $NextLocations

    # Clear Next location array to avoid folder creation of previous levels
    $NextLocations = @()
}
# Close status notification
Write-Progress -Activity "Processing $($ItemProcess)%" -Status "Site '$($Column)"

Write-Host -b Green "Finished creating folders!"
```