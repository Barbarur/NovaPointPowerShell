#Sync #Troubleshoot

<br>

## Close and Open OneDrive App (The classic)

Right-click on OneDrive icon in the notification bar. Select _Close OneDrive._

Restart OneDrive again. You can find it on your _All Apps_ at the _Start._ Or typing OneDrive in the _Windows Search_

## Clean the cache

1. Navigate to:
> *C:\Users\{USERNAME}\AppData\Local\Microsoft\Office\16.0*

2. Delete everything inside that folder.
<br>
## Clean Office and OneDrive Credential

1. Open Credential Manager.

2. Select Windows Credentials.

3. Extend and Remove the _Microsoft Office_ and _OneDrive Cached_ Credentials
<br>
## Reset OneDrive

1. Open _Run_ by pressing _Windows Key + R_.

2. Type the below dialog and press
> *%localappdata%\Microsoft\OneDrive\onedrive.exe /reset*
> If Windows cannot find onedrive.exe on that location, try either these 2 locations instead:
>> *C:\Program Files\Microsoft OneDrive\onedrive.exe /reset*
>> *C:\Program Files (x86)\Microsoft OneDrive\onedrive.exe /reset*

3. The OneDrive icon logo will disappear for a minute and re-appear afterwards.

4. If OneDrive doesn’t re-start automatically. Open again _Run,_ write the below dialog and press
> *%localappdata%\Microsoft\OneDrive\onedrive.exe*
<br>
## Check and delete old app versions

1. Right-Click on the OneDrive icon at the notification bar -> Settings -> About 

2. Look at which is the version number of the OneDrive you are using. 

3. Then press Win + R to open the _Run._ 

4. Write _Regedit_  and press OK. 

5. Navigate to the below address:
> *Computer\HKEY_CURRENT_USER\SOFTWARE\Microsoft\OneDrive\*

6. You should see a folder with a name the same as the version number you checked before. If you see other folders with different version numbers, then delete those. Ignore any other folder with normal names, we only focus on the ones with numbers as name. 

7. Inside the remaining folder with version number as name, you will see 3 files. Focus on the _InstallPatch_  and _InstallPaths._ 

The data should be the same. If the _InstallPaths_ has more than one data. Double-click on _InstallPath_ to open the Edit String window, copy the _Value data,_ close the window. Double-click on the _InstallPaths_ and paste it in the Value data box. 

8. Close the _Registry Editor_.  
<br>
## Unlink your account

1. Right-Click on the OneDrive icon in your notification bar -> Settings -> Account -> _Unlink this PC_

2. Restart the computer

3. Re-link your account. Right-Click on the OneDrive icon in your notification bar -> Settings -> Account -> _Add an account_
<br>
## Check in a different network.

Possibly the issue is not with the OneDrive Desktop App, but with the network you are using blocking connection to certains IPs.
<br>
## Connect from a different device

Your device might have a issue itself not related with account syncing. Try to connect from another device and see if the issue reproduce.
<br>
## Review Sync Policy for OneDrive.

If you are using OneDrive for Business. If there is any restriction based on Managed Devices, Network, Doemain joined devices or any other policy that can limit the sync of OneDrive Desktop App.
<br>
## Check the number of files syncing

Currently there is a limitation on the number of files you can sync using OneDrive Desktop App. At the moment of writing for Personal OneDrive is 2500 and for Business and SharePoint is 300,000. You can check limitations on Microsoft documentation for double confirmation in case there has been any modifications from the moment of writing until now when you read this article.

Check _Sync_ section: [SharePoint limits](https://docs.microsoft.com/en-us/office365/servicedescriptions/sharepoint-online-service-description/sharepoint-online-limits).

Check _Number of items_ that can be synced or copied section: [Restrictions and limitations in OneDrive](https://support.microsoft.com/en-us/office/restrictions-and-limitations-in-onedrive-and-sharepoint-64883a5d-228e-48f5-b3d2-eb39e07630fa).

It is very possible you can exceed this limitation and not having problems before, as Microsoft provides higher thresholds in the system, which vary from one server to another. But don’t get surprised if you start getting issues at some point.

One way around is sync your documents by batch. Selecting folders with a number of items below this limitation, sync the folder and when it finishes sync another one. This is not 100% bulletproof, as the app needs to map all documents and folders looking for changes and at some point it might bring problems again. But this is less common.

Also, if you have a big volume of documents, even if they are no heavy, it is better practice to use the function Save space and download files as you use them, which is available only for windows, but I imagine it eventually will arrive to Mac.

## Collect OneDrive Sync Logs

1. Navigate to: _%localappdata%\Microsoft\OneDrive\<CurrentInstalledVersion>_

If you cannot find the folder named as the current installed version, check the below locations

-   _C:\Program Files\Microsoft OneDrive\_
-   _C:\Program Files (x86)\Microsoft OneDrive\_

2. Search for the file **_CollectSyncLogs.bat_** and run it.

3. CollectSyncLogs.bat would open a terminal window while collecting the logs, and it will close it automatically once the operation finish.

4. Once the operation finish, you will find a .CAB file on the desktop.

5. You can create a ticket with Microsoft Support to help on analyzing these logs.

<br>
<br>
#### Reference
[Reset OneDrive - Microsoft Support](https://support.microsoft.com/en-us/office/reset-onedrive-34701e00-bf7b-42db-b960-84905399050c)
