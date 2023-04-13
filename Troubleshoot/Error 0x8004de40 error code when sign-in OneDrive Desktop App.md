#ErrorCode #PnP #PowerShell #Troubleshoot 

<br>

There are several ways to solve this issue based on the operative system used

## Windows 10

Use [Crypo IIS](https://www.nartac.com/Products/IISCrypto/) software to adjust the Ciphers.

<br>

## Windows 7

Few possibilities to solve this issue.

1. Use [Easy Fix Tool](https://download.microsoft.com/download/0/6/5/0658B1A7-6D2E-474F-BC2C-D69E5B9E9A68/MicrosoftEasyFix51044.msi) to add TLS Secure Protocol registry keys automatically.

2. Enable TLS 1.2 as default secure protocols in WinHTTP in Windows, by Windows Update

-   Click **Start**, type update in the search box, in the list of results, click **Windows Update**.
-   In the details pane, click **Check for updates**, and then wait while Windows looks for the latest updates for your computer.
-   If you see a message telling you that important or optional updates are available, or telling you to review important or optional updates, click the message to view the updates to install.
-   In the list, select the check box for the updates that you want to install, click **OK**, and then click **Install updates**.

3. Enable TLS 1.2 as default secure protocols in WinHTTP in Windows, by Microsoft Update Catalog

-   To get the stand-alone package for this update, go to the [Microsoft Update Catalog](http://catalog.update.microsoft.com/v7/site/search.aspx?q=kb3140245) website.

<br>

## Windows 2012R2

Few possibilities to solve this issue.

1. Use [Easy Fix Tool](https://download.microsoft.com/download/0/6/5/0658B1A7-6D2E-474F-BC2C-D69E5B9E9A68/MicrosoftEasyFix51044.msi) to add TLS Secure Protocol registry keys automatically.

2. Disable DHE cipher suites

-   Open Registry Editor.
-   Access key exchange algorithm settings by navigating to the following registry location:HKEY_LOCAL_MACHINE\SYSTEM\CurrentControlSet\Control\SecurityProviders\SCHANNEL\KeyExchangeAlgorithms
-   Select the **Diffie-Hellman** sub key (if it does not exist, then create it).
-   Set the **Enabled** DWORD registry value to **0** (if it does not exist, then create it).
-   Exit Registry Editor.  

3. Use [Crypo IIS](https://www.nartac.com/Products/IISCrypto/) software to adjust the Ciphers.

4. Update through Windows Update

-   Swipe in from the right edge of the screen, and then tap **Search**. Or, if you are using a mouse, point to the lower-right corner of the screen, and then select **Search**.
-   In the search box, type _Windows Update_, and then tap or select **Windows Update**.
-   In the details pane, select **Check for updates**, and then wait while Windows looks for the latest updates for your computer.
-   If you see a message telling you that important or optional updates are available, or telling you to review important or optional updates, select the message to view the updates to install.
-   In the list, select the check box for the updates that you want to install under **Important** or **Optional**, and then tap or select **Install**.

<br>
<br>

#### References

-   [Error Code 0x8004de40 when signing in to OneDrive](https://docs.microsoft.com/en-us/sharepoint/troubleshoot/administration/error-0x8004de40-in-onedrive)
-   [Disable DHE cipher suites](https://docs.microsoft.com/en-us/security-updates/securitybulletins/2015/ms15-055#workarounds)
-   [How to get an update through Windows Update](https://docs.microsoft.com/en-us/troubleshoot/windows-client/deployment/get-updates-through-windows-update)
-   [Update to enable TLS 1.1 and TLS 1.2 as default secure protocols in WinHTTP in Windows](https://support.microsoft.com/en-us/topic/update-to-enable-tls-1-1-and-tls-1-2-as-default-secure-protocols-in-winhttp-in-windows-c4bd73d2-31d7-761e-0178-11268bb10392)
-   [Support for TLS System Default Versions included in the .NET Framework 3.5 on Windows 8.1 and Windows Server 2012 R2](https://support.microsoft.com/en-us/topic/support-for-tls-system-default-versions-included-in-the-net-framework-3-5-on-windows-8-1-and-windows-server-2012-r2-499ff5ef-a88a-128b-c639-ed038b7d2d5f)
-   [Microsoft Update Catalog](http://catalog.update.microsoft.com/v7/site/search.aspx?q=kb3140245)