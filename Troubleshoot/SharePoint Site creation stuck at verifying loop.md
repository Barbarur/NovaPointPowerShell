#Troubleshoot #SharePointOnline #SiteCollection #PowerShell #SPOService 

<br>

The main root cause if this issue Redirect. The redirect might happens because:

1.  [Change the SharePoint Site address](https://docs.microsoft.com/en-us/sharepoint/change-site-address).
2.  [Swapping a Site](https://docs.microsoft.com/en-us/sharepoint/modern-root-site#replace-your-root-site).
3.  [Moving Site to a different Geo location](https://docs.microsoft.com/en-us/microsoft-365/enterprise/move-sharepoint-between-geo-locations?view=o365-worldwide).

In all the above cases a new URL is created and the old one is kept for redirection.

You can check the existing redirects on your tenant by running the below command using PowerShell and the [SharePoint Online Management Shell](https://novacato.com/get-powershell-ready-before-start-to-work/).
```powershell
Get-SPOSite -Template REDIRECTSITE#0
```

If you find the name you are using for the new site is matching one of the redirect URL, you can delete that URL using the command below.

Check with the Site owners before to ensure any connection they are using with the site is updated with the new URL.

```powershell
Remove-SPOSite -Identity https://contoso.sharepoint.com/sites/site1 -NoWait
```

<br>

#### Reference

-   [Manage site redirects](https://docs.microsoft.com/en-us/sharepoint/manage-site-redirects)