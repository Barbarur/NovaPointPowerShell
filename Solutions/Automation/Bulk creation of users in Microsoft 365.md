#Automation #PowerShell #Msol #CSV #User

<br>

## Prepare CSV with user information

| UPN | DN | FN | LN |
| --- | --- | --- | --- |
| User1@UPN.com | User1DisplayName | User1FirtName | User1Lastname |
| User2@UPN.com | User2DisplayName | User2FirtName | User2Lastname |
| User3@UPN.com | User3DisplayName | User3FirtName | User3Lastname |

<br>

## Run

```powershell
# Define Parameters
$CSVFile = "$Env:USERPROFILE\Documents\UserList.csv"

# Connect to Microsoft 365
Connect-MsolService

# Import User List
$CSVData = Import-CSV $CSVFile

# Loop through each user
$ItemCounter = 0 
ForEach($Row in $CSVData){
    # Status notification
    $ItemCounter++
    $ItemProcess = [math]::Round($ItemCounter/$CSVData.Count*100,1)
    Write-Progress -PercentComplete $ItemProcess -Activity "Processing $($ItemProcess)%" -Status "User: '$($Row.UPN)"

    # Create new user
    New-MsolUser -UserPrincipalName $Row.UPN -DisplayName $Row.DN -FirstName $Row.FN -LastName $Row.LN
}
# Close status notification
Write-Progress -Activity "Processing $($ItemProcess)%" -Status "Site '$($Row.URL)"

Write-Host -b Green "Finished adding new users to MS365!"
```
