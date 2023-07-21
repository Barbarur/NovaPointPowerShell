#Automation #PowerShell #PnP #CSV

<br>


```powershell
#Config Variables
$SiteURL = "https://<Domain>.sharepoint.com/sites/<SiteName>"
$ListTitle = "Doc01"
$FileRelativeURL = "/sites/<SiteName>/<LibraryName>/<FolderName>/<SubFolderName>"
$DownloadPath ="C:\Temp"

#$Cred = Get-Credential

Try {
    #Connect to PnP Online
    Connect-PnPOnline -Url $SiteURL -Credentials $Cred

    $Items = Get-PnPListItem -List $ListTitle -FolderServerRelativeUrl $FileRelativeURL -PageSize 2000
    
    ForEach($Item in $Items){
        If($Item.FileSystemObjectType -eq "Folder") {Continue}
        write-host -f Cyan $Item["FileRef"]
        write-host -f yellow $Item.FieldValues.FileLeafRef
        Get-PnPFile -Url $Item["FileRef"] -Path $DownloadPath -FileName $Item["FileLeafRef"] -AsFile
    }
}
catch {
    write-host "Error: $($_.Exception.Message)" -foregroundcolor Red
}
```