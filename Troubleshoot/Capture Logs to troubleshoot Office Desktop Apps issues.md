#Troubleshoot #Excel #Word #PowerPoint

<br>

## Installations needed

1.  Download and Install [Fiddler](https://www.telerik.com/fiddler).
2.  Download and Install [Process Monitor](https://docs.microsoft.com/en-us/sysinternals/downloads/procmon).
3.  Download and Install [ProcDump](https://docs.microsoft.com/en-us/sysinternals/downloads/procdump).

<br>

## Preparation

1.  Open **Fiddler**.
    -   Navigate to _Tools_ > _Options_ > _HTTPS_ > Enable _Capture_ and _Decrypt_.
    -   Click on _File_ and ensure _Capture Traffic_ is enabled.
2.  Open **Process Monitor**.
3.  Open the **Command Promt** and run the command below, change the app name according to your situation. A pop-up window will open, do not close it.
    -   procdump.exe –w -t -accepteula -ma powerpoint.exe
        
4.  Open PowerShell and run the below commands:
```powershell
reg add HKCU\Software\Microsoft\Office\16.0\Common\Logging /v MsoEtwTracingEnabled /t REG_DWORD /d 1
reg add HKCU\Software\Microsoft\Office\16.0\Common\Logging /v EnableLogging /t REG_DWORD /d 1
```

<br>

## Reproduce the issue

Open the affected file and reproduce the issue.

<br>

## Save logs

1.  Go to Fiddler > _File_ > _Save_ > _All sessions_.
2.  Go to Process Monitor > _File_ > _Save_. Select _Events displayed using current filter_ and choose the Path and click OK. Zip the files and name it ‘_ProcessMonitorLogs_DATE_‘.
3.  Go to the pop-up window, opened with the Command Promt command before, and save the generated logs. Zip them and name it ‘_DumpLogs_DATE_“.
4.  Navigate to _C:\\Users\\USERNAME\\AppData\\Local\\Temp,_ you would see the files named ‘_MachineName-Date-time.log’_. Collect those files and Zip them and name it ‘ClientLogsv1_DATE‘.
5.  Navigate to _C:\\Users\\USERNAME\\AppData\\_Loca\l\_Temp\\Diagnostics,_ you would see a folder for each different application, Zip the folder of the application you have used during the test and name it ‘ClientLogsv2_DATE‘.
6.  Open PowerShell and run the below command to stop collecting logs:
```powershell
reg delete HKCU\Software\Microsoft\Office\16.0\Common\Logging /v MsoEtwTracingEnabled
reg delete HKCU\Software\Microsoft\Office\16.0\Common\Logging /v EnableLogging
```
7.  Navigate to _C:\Users\USERNAME\AppData\Local\Microsoft\OneDrive\CURRENTVERSION_ open the file CollectSyncLogs.bat. A sync logs file will be generated on the Desktop. Zip it and name it as ‘OneDriveSyncLogs_DATE‘.