#Report #AzureAD #PowerShell #ProfilePicture

<br>

## Option 1: Check the profile picture of all users in Azure AD

```powershell
#Config Parameters
$ReportOutput = "C:\Temp\UserPictureReport.csv"

Connect-AzureAD

#Array to store results
$Results = @()

#Get the user's profile picture
$ListUsers = Get-AzureADUser -All $true


ForEach($User in $ListUsers){
    $UserName = $User.DisplayName
    Try{
        $ProfilePics = Get-AzureADUserThumbnailPhoto -ObjectId $user.ObjectId
         
        If($ProfilePics -ne $null){
             
            ForEach($ProfilePic in $ProfilePics){
                Write-Host "User $UserName has profile picture in Azure AD" -ForegroundColor Green
                $Results += New-Object PSObject -Property ([ordered]@{
                    UserName           = $User.DisplayName
                    UserID             = $User.ObjectId
                    UPN                = $User.UserPrincipalName
                    UserEmail          = $User.Mail
                    UserPicture        = "Yes"
                })
            }
        }
            
    } 
    Catch{
        Write-Host "User $userName does NOT have a profile picture in Azure AD"  -ForegroundColor Red
        $Results += New-Object PSObject -Property ([ordered]@{
            UserName           = $User.DisplayName
            UserID             = $User.ObjectId
            UPN                = $User.UserPrincipalName
            UserEmail          = $User.Mail
            UserPicture        = "Null"
        })
    }
}

$Results | Export-Csv -Path $ReportOutput -NoTypeInformation
Write-host -f Green "File Size Report Exported to CSV Successfully!"
```

<br>

## Option 2: Check the profile picture of user in a list

Ensure the location of the User list _**row 2**_ and _objectID_ in **_row 16_** are correct before running the script.

```powershell
#Config Parameters
$CSVFile = "C:\Temp\UserList.csv"
$ReportOutput = "C:\Temp\UserPictureReport.csv"

$CSVData = Import-CSV $CSVFile
Connect-AzureAD

#Array to store results
$Results = @()

Write-host -f Yellow "Total Number of Users in the List:"$CsVData.Count

$ItemCounter = 0 
ForEach($Row in $CSVData){
    $User = Get-AzureADUser -ObjectId $Row.'UserName'
    Try{
        $ProfilePics = Get-AzureADUserThumbnailPhoto -ObjectId $User.ObjectId
         
        If($ProfilePics -ne $null){
             
            ForEach($ProfilePic in $ProfilePics){
                Write-Host "User" $User.DisplayName "has profile picture in Azure AD" -ForegroundColor Green
                $Results += New-Object PSObject -Property ([ordered]@{
                    UserName           = $User.DisplayName
                    UserID             = $User.ObjectId
                    UPN                = $User.UserPrincipalName
                    UserEmail          = $User.Mail
                    UserPicture        = "Yes"
                })
            }
        }
            
    } 
    Catch{
        Write-Host "User" $User.DisplayName "does NOT have a profile picture in Azure AD"  -ForegroundColor Red
        $Results += New-Object PSObject -Property ([ordered]@{
            UserName           = $User.DisplayName
            UserID             = $User.ObjectId
            UPN                = $User.UserPrincipalName
            UserEmail          = $User.Mail
            UserPicture        = "Null"
        })
    }
    Write-Progress -PercentComplete ($ItemCounter / ($CsVData.Count) * 100) -Activity "Processing Item $ItemCounter of $($CsVData.Count)" -Status "Getting data from User $($User.DisplayName)"
}

$Results | Export-Csv -Path $ReportOutput -NoTypeInformation
Write-host -f Green "File Size Report Exported to CSV Successfully!"
```