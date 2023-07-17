#Troubleshoot #ErrorMessage  #PowerShell #PnP #RetentionPolicy 

<br>

## Root cause #1

The Site is under a Retention Policy or a Data Loss Prevention Policy. Check this by navigating to _SharePoint Admin Center – > Sites -> Active Sites -> Click on the affected site_. If there is a retention policy applied to that site you will se a message as below picture.

<br>

Another option to check if there is a retention policy is to visit the URL:

_https://XXX-my.sharepoint.com/personal/{USERNAME_DOMAIN__COM}/_layouts/15/viewlsts.aspx?view=14_

or

_https://XXX.sharepoint.com/sites/{SITENAME}/_layouts/15/viewlsts.aspx?view=14_

You will be driven to the Site contents page. Check if there is a folder called _Preservation Hold Library_.

<br>

### Root cause #1 . Solution #1

Use a PowerShell script to delete files inside of the folder one by one.

```powershell
#Define Parameters
$SiteURL = "https://DOMAIN-my.sharepoint.com/sites/SITENAME"
$ListName ="Documents"
$FolderServerRelativeURL = "/sites/SITENAME/Shared Documents/FOLDER"
  
Try {
    #Connect to PnP Online
    Connect-PnPOnline -Url $SiteURL -UseWebLogin
       
    #Get All Items from Folder in Batch
    $ListItems = Get-PnPListItem -List $ListName -FolderServerRelativeUrl $FolderServerRelativeURL -PageSize 2000 | Sort-Object ID -Descending
    Write-host "Total Number of Items Found:"$ListItems.count
   
    #Powershell to delete all files from a folder
    ForEach ($Item in $ListItems)
    {
        Remove-PnPListItem -List $ListName -Identity $Item.Id -Recycle -Force
        Write-host "Removed File:"$Item.FieldValues.FileRef
    }
}
Catch {
    write-host "Error: $($_.Exception.Message)" -foregroundcolor Red
}
```

<br>

### Root cause #1 . Solution #2

Variation of the previous script for deletion of all files in the folder one by one.

```powershell
#Define Parameters
$SiteURL = "https://DOMAIN-my.sharepoint.com/sites/SITENAME"
$ListName ="Documents"
$FolderServerRelativeURL = "/sites/SITENAME/Shared Documents/FOLDER"

Try {
    #Connect to PnP Online
    Connect-PnPOnline -Url $SiteURL -UseWebLogin
       
    #Get All Items from Folder in Batch
    $ListItems = Get-PnPListItem -List $ListName -PageSize 2000 | Sort-Object ID -Descending
      
    #Get List Items from the folder
    $ItemsFromFolder = $ListItems | Where {$_.FieldValues["FileDirRef"] -match $FolderServerRelativeURL }
    
    Write-host -f Cyan "Total Number of Items in the Site:"$ListItems.count
    Write-host -f Cyan "Total Number of Items in the location:"$ItemsFromFolder.count
    
    $ItemCounter = 0 

    #Powershell to delete all files from a folder
    ForEach ($Item in $ItemsFromFolder)
    {
        $ItemCounter++
        $ItemProcess = [math]::Round($ItemCounter/$ItemsFromFolder.Count*100,1)
        Write-Progress -PercentComplete $ItemProcess -Activity "Processing $($ItemProcess)%" -Status "Item: $($Item.DisplayName)"
        $Msg = "Removing File '{0}'... " -f $Item.Title
        Write-Host -f Yellow $Msg -NoNewline

        Remove-PnPListItem -List $ListName -Identity $Item.Id -Recycle -Force
        Write-Host -f Green " COMPLETED"
        Write-host -f DarkYellow "at"$Item.FieldValues["FileRef"]
    }
}
Catch {
    write-host "Error: $($_.Exception.Message)" -foregroundcolor Red
}
```

<br>

## Root cause #2

If you have enabled a Check Out on you Library, one of the documents inside the folder or sub-folder might be checked-out. This can also trigger the error message.

First check if you have the Check Out enabled by navigating to the affected Document Library -> Settings -> Library Settings -> Versioning Settings -> Require Check Out.

### Root cause #2 . Solution #1

Check In all the files inside the folder you want to delete. [Visit this article for more information about how to do it](obsidian://open?vault=NovaPointPowerShell&file=Solutions%2FAutomation%2FCheck-In%20all%20files%20in%20a%20specific%20location).

<br>

<br>
