#Troubleshoot #Policy #Sync

<br>

1. Go to [SharePoint Admin Center](https://admin.microsoft.com/sharepoint) with admin permissions. Navigate to Settings -> OneDrive –> Sync
2. Enable _Allow syncing only for computers joined to specific domains_
3. Add the [GUID](https://docs.microsoft.com/en-us/powershell/module/activedirectory/get-addomain?view=windowsserver2019-ps) for each domain you want to give exclusive access. The domain [GUID](https://docs.microsoft.com/en-us/powershell/module/activedirectory/get-addomain?view=windowsserver2019-ps) is of your on-premises Active Directory.
4. If you do not know your domain [GUID](https://docs.microsoft.com/en-us/powershell/module/activedirectory/get-addomain?view=windowsserver2019-ps); use **PowerShell ISE** as Adminstrator to connect to your On-Premises Active Directory and the run the command **Get-ADDomain**. Look for **ObjectGUID**. That is what you have to copy in your settings.
5. You can add more than 1 domain. Just need to separate 1 domain per line.
6. Save the changes.
7. Wait for some changes to take place