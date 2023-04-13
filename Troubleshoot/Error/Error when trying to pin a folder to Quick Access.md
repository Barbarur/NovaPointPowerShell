#ErrorMessage #Troubleshoot #Windows #FileExplorer #QuickAccess

<br>

Open the **Command Prompt** and run the below commands:
```powershell
del /F /Q %APPDATA%\Microsoft\Windows\Recent\*
```

```powershell
del /F /Q %APPDATA%\Microsoft\Windows\Recent\AutomaticDestinations\*
```

```powershell
del /F /Q %APPDATA%\Microsoft\Windows\Recent\CustomDestinations\*
```
