#Report #SharePointOnline  #PowerShell #PnP #Everyone #EveryoneExceptExternalUsers #SiteCollection

<br>

```powershell
#Define Parameters
$AdminSiteURL= "https://DOMAIN-admin.sharepoint.com"
$ReportOutput = "C:\Temp\EveryoneReport.csv"

#Connect to PnP Online
Connect-PnPOnline -Url $AdminSiteURL -interactive

#Get owners of each Site
$Results = @()
$ItemCounter = 0 

#Get all Sites and itinerate
$TenantSites = Get-PnPTenantSite | Where{ ($_.Title -notlike "" -and $_.Template -notlike "*Redirect*" -and $_.Url -notlike "*my.sharepoint.com*") }
ForEach($TenantSite in $TenantSites){

    #Status notification
    $ItemCounter++
    $ItemProcess = [math]::Round($ItemCounter/$TenantSites.Count*100,1)
    Write-Progress -PercentComplete ($ItemCounter / ($TenantSites.Count) * 100) -Activity "Processing $($ItemProcess)%" -Status "Site '$($TenantSite.Title)"

    Connect-PnPOnline -Url $TenantSite.url -interactive
    $WebRoles = Get-PnPWeb -Includes RoleAssignments
    ForEach ($SiteRoleAssignment in $WebRoles.RoleAssignments){
        #Get the Permission Levels assigned and Member
        Get-PnPProperty -ClientObject $SiteRoleAssignment -Property RoleDefinitionBindings, Member
     
        #Get the Permission Levels assigned
        $SitePermissionLevels = ($SiteRoleAssignment.RoleDefinitionBindings | Select -ExpandProperty Name | Where { ($_ -ne "Limited Access") -and ($_ -ne "Web-Only Limited Access")} ) -join ","
        If($SitePermissionLevels.Length -eq 0 -or $SiteRoleAssignment.Member.Title -clike '*Limited Access System Group*') {Continue}

        $SitePermissionType = $SiteRoleAssignment.Member.PrincipalType
            
        If($SitePermissionType -eq "SharePointGroup") {
            $GroupMembers = Get-PnPGroupMember -Identity $SiteRoleAssignment.Member.Title
            ForEach($GroupMember in $GroupMembers){
                If($GroupMember.Title -eq "Everyone" -or $GroupMember.Title -eq "Everyone except external users"){
                    $Results += New-Object PSObject -Property ([ordered]@{
                        SiteName               = $TenantSite.Title
                        SiteURL                = $TenantSite.url
                        From                   = $SiteRoleAssignment.Member.Title
                        Group                  = $GroupMember.Title
                        PermissionLevels       = $SitePermissionLevels
                    })
                }
            }
        }
        Else{
            If($SiteRoleAssignment.Member.Title -eq "Everyone" -or $SiteRoleAssignment.Member.Title -eq "Everyone except external users"){
                $Results += New-Object PSObject -Property ([ordered]@{
                    SiteName               = $TenantSite.Title
                    SiteURL                = $TenantSite.url
                    From                   = "Direct Permission"
                    Group                  = $SiteRoleAssignment.Member.Title
                    PermissionLevels       = $SitePermissionLevels
                })
            }
        }
    }
}
#Close status notification
Write-Progress -Activity "Processing $($ItemProcess)%" -Status "Site '$($TenantSite.URL)"

#Export the results to CSV
If (Test-Path $ReportOutput) { Remove-Item $ReportOutput }
$Global:Results | Export-Csv -Path $ReportOutput -NoTypeInformation
Write-host -b Green "Report Generated Successfully!"
Write-host -f Green $ReportOutput
```