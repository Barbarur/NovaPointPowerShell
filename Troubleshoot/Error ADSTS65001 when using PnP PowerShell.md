#ErrorCode #PnP #PowerShell #Troubleshoot 

<br>

In some cases when using PnP Powershell to connect SharePoint Error ADSTS65001, might pop-up. In order to solve that issue, you need to run the below commands before running your script.

```
# Define Parameters
$SiteURL = "https://<DOMAIN>-admin.sharepoint.com"

# Connect PnP Online
Connect-PnPOnline -Url $SiteURL -PnPManagementShell
Disconnect-PnPOnline
Connect-PnPOnline $SiteURL
Disconnect-PnPOnline
```

<br>
<br>
#### Reference
[Error ADSTS65001](https://github.com/pnp/sp-starter-kit/issues/436#issuecomment-694936145)
