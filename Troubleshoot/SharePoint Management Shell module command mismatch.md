#Troubleshoot #PowerShell #SPOService 

<br>

1. Uninstall SharePoint Management Shell.

2. If you cannot uninstall it from the program list, then Download and Install the latest Version of [SharePoint Management Shell](https://www.microsoft.com/en-us/download/details.aspx?id=35588).

3. Open PowerShell as Admin and run the below command. This would Uninstall any module installed on the device.

```powershell
Uninstall-Module -Name Microsoft.Online.SharePoint.PowerShell -AllVersions
```

4. Once the process finishes, use the command below to ensure there is no Module installed. The result should be empty.

```powershell
Get-Module -Name Microsoft.Online.SharePoint.PowerShell -ListAvailable | Select Name,Version
```

6. Close PowerShell ISE and any other PowerShell currently open.

7. Navigate to _C:\Windows\Microsoft.NET\assembly\GAC_MSIL_

8. Delete any folder which is “_Microsoft.Sharepoint…._“

9. Open PowerShell ISE as Admin and use the below command to install the latest SharePoint Module.

```powershell
Install-Module -Name Microsoft.Online.SharePoint.PowerShell
```

10. One the Installation finished, close PowerShell ISE and open it again as Admin.

11. Run the below command to ensure the SharePoint Module commands are correctly exported. Under _ExportedCommands_ you should see the commands, it shouldn’t be empty.

```powershell
Get-Module -ListAvailable *sharepoint*
```