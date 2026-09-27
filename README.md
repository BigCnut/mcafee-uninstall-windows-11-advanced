# HowTo Deinstall McAfee on Windows 11 when normal uninstall and MCPR fail

## Root cause
The affected Windows 11 installation had a damaged McAfee removal path rather than a simple stale Apps entry.

The normal McAfee uninstaller no longer worked, and MCPR reported incomplete uninstallation because its cleanup component could not be loaded. At the same time, substantial McAfee components were still installed and active: protected services, kernel drivers, SystemCore/VSCore/ModuleCore components, scheduled tasks, and McAfee registry registrations.

The decisive blockers were McAfee self-protection and protected services/drivers. Normal Windows administration could not reliably stop or remove those components. The successful resolution therefore required working offline from Windows Recovery Environment (WinRE), temporarily disabling the relevant protection/driver mechanisms, allowing McAfee's own VSCore removal component from the MCPR package to remove the protected core, and then removing the remaining registrations/files offline.

The procedure below is derived from the successful path and does not include the explorative steps that lead to the successful path. MCPR package paths and the product GUID are version-dependent and must be taken from the current MCPR package; the example values from the original PC are deliberately not reproduced.

---

## Disclaimer — Use at Your Own Risk
This procedure is provided for informational and troubleshooting purposes only and is intended for use by experienced users who understand the risks associated with system-level modifications. You perform all steps entirely at your own risk. The author provides no guarantee that the procedure will work in every environment and assumes no responsibility or liability for any damage, data loss, system instability, loss of functionality, security issues, or other direct or indirect consequences resulting from the use or misuse of this information. Before proceeding, ensure that you have appropriate backups and recovery options. If you are not confident that you understand a particular step or its potential consequences, do not perform it and seek assistance from a qualified professional.

## Warning — Advanced system-level procedure
This procedure is intended only for experienced Windows administrators or users with a thorough understanding of Windows internals, the Windows Registry, services, drivers, WinRE, and elevated command-line operations. It should not be attempted by inexperienced users.

The steps in this document go substantially beyond a normal software uninstall. They involve modifying the offline Windows Registry, disabling and renaming kernel-mode drivers, changing protected-service configuration, removing services and driver registrations, and manually deleting files and registry entries. These operations can affect components that are essential to Windows startup and system operation.

### Potential consequences
An incorrect command, an incorrect registry modification, or deleting the wrong file or registry entry can result in, among other things:
* Windows failing to boot or becoming stuck in a recovery/repair loop.
* Loss of access to Windows services or other system components.
* Blue screens or other system instability caused by incorrectly modifying driver configuration.
* McAfee or other security components being left in an inconsistent or partially removed state.
* Windows Security or Microsoft Defender failing to register or operate correctly.
* Loss of network connectivity, firewall functionality, or other security-related functionality.
* Corruption or unintended modification of the Windows Registry.
* Loss of applications, configuration, or user data in severe cases.
* A system that can only be recovered by using System Restore, registry/hive backups, Windows Recovery Environment, or Windows reinstallation.

Registry and driver modifications are particularly dangerous because a seemingly minor mistake can prevent Windows from starting normally. An operation performed against the wrong registry hive, registry key, service, driver, or filesystem path can affect Windows itself rather than merely McAfee.

---

## Before proceeding
Do not rely solely on the restore point created as part of this procedure. Make a separate, verified backup of important personal data before making any system-level changes. If possible, ensure that you have a recovery method available, such as Windows installation/recovery media and the credentials or recovery information required to access the system.

The registry hive backups described in this document should also be treated as an additional recovery measure, not as a guarantee that the system can be restored to its previous state.

Before executing any command, carefully verify:
1. That the system is actually running Windows 11 and that the correct Windows installation and system volume have been identified in WinRE.
2. That the registry hive being loaded is the hive belonging to the intended Windows installation.
3. That every registry key, service, driver, and file being modified or removed is actually associated with McAfee and is not a Windows or third-party component.
4. That commands referring to drive letters use the drive letters currently assigned in WinRE; these can differ from those used during normal Windows operation.
5. That paths, service names, driver names, and registry values have been copied or entered exactly as intended.
6. That the MCPR package and its components are obtained from a trustworthy, current source and are appropriate for the installed McAfee product.

Do not improvise based on similar-looking registry entries, services, or driver names. If a step produces an unexpected result, stop and investigate it rather than continuing with subsequent cleanup operations.

## Recovery considerations
Keep the recovery options available before starting the procedure. If Windows becomes unbootable, do not immediately continue deleting or modifying registry entries in an attempt to repair the problem. First determine which change caused the failure and use an appropriate recovery mechanism, such as System Restore or restoration of the backed-up registry hives.

If you cannot confidently interpret a registry key, service configuration, driver registration, WinRE drive assignment, or command-line result, stop the procedure and obtain assistance from someone experienced with Windows system administration.

## Security software considerations
Removing an antivirus product can temporarily leave the computer without its intended third-party protection. The final verification is therefore important: Windows Security should report an appropriate active antivirus provider, and Microsoft Defender Antivirus should be correctly registered and operational where it is intended to provide protection.

Do not consider the procedure complete merely because the McAfee application directory has disappeared. Verify the services, drivers, scheduled tasks, registry registrations, filesystem components, and Windows Security registration as described in the final verification section.

## Responsibility and suitability
This procedure is provided as an advanced troubleshooting and recovery procedure, not as a routine uninstall method. The fact that a command is shown in this document does not make it safe to execute without understanding what it changes.

If the standard McAfee uninstaller and MCPR fail, the preferred course for users without the necessary system-administration knowledge is to stop here and seek qualified assistance rather than proceeding with offline registry and driver modifications.

Proceed only if you understand the commands you are executing, have appropriate backups and recovery options, and accept the possibility that an error may require substantial system recovery or, in the worst case, a Windows reinstallation.

## Support
This document is provided on an as-is, self-service basis. The author does not provide individual support, troubleshooting, or assistance with applying the procedure, interpreting errors, or recovering from problems resulting from its use. Please do not contact the author requesting support or asking for assistance with individual cases. If you encounter an unexpected result or are unsure how to proceed, stop and consult a qualified Windows administrator or other appropriately experienced professional.

---

## Summary
1. Create a restore point and backups of the Windows `SYSTEM` and `SOFTWARE` registry hives.
2. In WinRE, load the offline registry hives and disable McAfee self-protection.
3. Disable the McAfee kernel driver `mfehidk` offline and rename its driver file so it cannot load on the next boot.
4. In normal Windows, remove the protected-service barrier from `mfefire`.
5. Use the current MCPR package's `mfehidin.exe` VSCore uninstall component. If it reports that an authorization value is required, create the authorization value and rerun it.
6. Reboot when the McAfee cleanup reports that operations are deferred until reboot.
7. If residual McAfee services, scheduled tasks, drivers, files, or uninstall registrations remain, remove those residual registrations and files from a single WinRE session.
8. Unload the offline registry hives and reboot.
9. Verify that no McAfee services, scheduled tasks, filesystem filter drivers, driver registrations, McAfee program/data directories, or antivirus registrations remain. Confirm that Windows Defender is the registered antivirus.

---

## Detailed steps

### 1. Prepare a recovery point and registry backups
Open an elevated Command Prompt.

Create a restore point:
```powershell
powershell -NoProfile -Command "Checkpoint-Computer -Description 'Before McAfee removal' -RestorePointType 'MODIFY_SETTINGS'"
```

Create a backup directory and back up the two relevant registry hives:
```cmd
mkdir C:\McAfeeBackup
copy C:\Windows\System32\config\SYSTEM C:\McAfeeBackup\SYSTEM.bak
copy C:\Windows\System32\config\SOFTWARE C:\McAfeeBackup\SOFTWARE.bak
```

---

### 2. Enter WinRE and disable McAfee self-protection
From an elevated Command Prompt:
```cmd
shutdown /r /o /t 0
```

Select **Troubleshoot** → **Advanced options** → **Command Prompt**.

Verify that the Windows installation is mounted as `C:`. If it is not, substitute the correct Windows volume letter in all following commands.

Load the offline hives:
```cmd
reg load HKLM\OFFLINE_SYSTEM C:\Windows\System32\config\SYSTEM
reg load HKLM\OFFLINE_SOFTWARE C:\Windows\System32\config\SOFTWARE
```

Disable McAfee self-protection:
```cmd
reg add "HKLM\OFFLINE_SOFTWARE\McAfee\AVSolution\MCSHIELDGLOBAL\GLOBAL" /v enableselfprotection /t REG_DWORD /d 0 /f
```

Verify:
```cmd
reg query "HKLM\OFFLINE_SOFTWARE\McAfee\AVSolution\MCSHIELDGLOBAL\GLOBAL" /v enableselfprotection
```
*(The value should be `0x0`.)*

Unload the hives:
```cmd
reg unload HKLM\OFFLINE_SOFTWARE
reg unload HKLM\OFFLINE_SYSTEM
```

Reboot normally:
```cmd
wpeutil reboot
```

---

### 3. Disable and neutralize the McAfee kernel driver
Enter WinRE again:
```cmd
shutdown /r /o /t 0
```

Load the SYSTEM hive:
```cmd
reg load HKLM\OFFLINE_SYSTEM C:\Windows\System32\config\SYSTEM
```

Back up the specific driver registration:
```cmd
reg export "HKLM\OFFLINE_SYSTEM\ControlSet001\Services\mfehidk" C:\McAfeeBackup\mfehidk.reg
```

Disable the driver:
```cmd
reg add "HKLM\OFFLINE_SYSTEM\ControlSet001\Services\mfehidk" /v Start /t REG_DWORD /d 4 /f
```

Rename the driver file so it cannot load during the next boot:
```cmd
ren C:\Windows\System32\drivers\mfehidk.sys mfehidk.sys.bak
```

Unload the hive:
```cmd
reg unload HKLM\OFFLINE_SYSTEM
```

Reboot normally:
```cmd
wpeutil reboot
```

---

### 4. Remove the protected-service barrier from mfefire
In normal Windows, verify the service protection setting:
```cmd
reg query "HKLM\SYSTEM\CurrentControlSet\Services\mfefire" /v LaunchProtected
```

If it is still protected (`0x3` in the case documented here), enter WinRE:
```cmd
shutdown /r /o /t 0
```

Load the SYSTEM hive:
```cmd
reg load HKLM\OFFLINE_SYSTEM C:\Windows\System32\config\SYSTEM
```

Disable protection for this service in the offline configuration:
```cmd
reg add "HKLM\OFFLINE_SYSTEM\ControlSet001\Services\mfefire" /v LaunchProtected /t REG_DWORD /d 0 /f
```

Unload the hive:
```cmd
reg unload HKLM\OFFLINE_SYSTEM
```

Reboot normally:
```cmd
wpeutil reboot
```

Verify:
```cmd
reg query "HKLM\SYSTEM\CurrentControlSet\Services\mfefire" /v LaunchProtected
sc query mfefire
```
*(`LaunchProtected` should be `0x0`, and the service should be controllable/stopped.)*

---

### 5. Run the McAfee VSCore removal component from the current MCPR package
Use the `mfehidin.exe` supplied by the current MCPR package. Do not reuse the temporary directory or product GUID from another PC.

The command has this form:
```cmd
"<MCPR_TEMP_DIR>\VS\vscore\latest\64\mfehidin.exe" -u -g <MCAFEE_PRODUCT_GUID> -l C:\mfehidin-vscore.log
```

If the command reports:
> `Operation not authorized. Create an authorization value <MCAFEE_PRODUCT_GUID> with 1 as REG_DWORD data to be authorized for this operation.`

Create the requested authorization value:
```cmd
reg add "HKLM\SOFTWARE\WOW6432Node\McAfee\SystemCore\install_auth" /v "<MCAFEE_PRODUCT_GUID>" /t REG_DWORD /d 1 /f
```

Then rerun the VSCore removal command, using a new log file:
```cmd
"<MCPR_TEMP_DIR>\VS\vscore\latest\64\mfehidin.exe" -u -g <MCAFEE_PRODUCT_GUID> -l C:\mfehidin-vscore2.log
```

Allow the operation to finish.

The successful run in the source conversation removed/de-configured major McAfee components, including protected service and SystemCore/VSCore references. It also reported that some operations were deferred until reboot.

---

### 6. Reboot when the cleanup requests it
If the McAfee cleanup reports that operations were deferred and that a reboot is required, reboot normally:
```cmd
shutdown /r /t 0
```

Do not repeat cleanup commands before that reboot.

---

### 7. Remove residual McAfee services, tasks, files, and uninstall registrations offline
If the cleanup has completed but residual McAfee components remain, perform the remaining cleanup from one WinRE session.

Enter WinRE:
```cmd
shutdown /r /o /t 0
```

Load both hives:
```cmd
reg load HKLM\OFFLINE_SYSTEM C:\Windows\System32\config\SYSTEM
reg load HKLM\OFFLINE_SOFTWARE C:\Windows\System32\config\SOFTWARE
```

Remove the residual McAfee service registrations identified in the source case:
```cmd
for %S in ("McAfee WebAdvisor" McAPExe McAWFwk mccspsvc ModuleCoreService PEFService) do @reg delete "HKLM\OFFLINE_SYSTEM\ControlSet001\Services\%~S" /f
```

Remove the residual McAfee scheduled-task files:
```cmd
del /f "C:\Windows\System32\Tasks\McAfee Remediation (Prepare)" "C:\Windows\System32\Tasks\McAfeeLogon" "C:\Windows\System32\Tasks\McAfee\McAfee Auto Maintenance Task Agent" "C:\Windows\System32\Tasks\McAfee\McAfee Idle Detection Task"
rmdir /q "C:\Windows\System32\Tasks\McAfee" 2>nul
```

Remove the remaining McAfee program/data trees:
```cmd
rmdir /s /q "C:\Program Files\McAfee\MSC"
rmdir /s /q "C:\Program Files\McAfee"
rmdir /s /q "C:\Program Files\Common Files\McAfee"
rmdir /s /q "C:\Program Files (x86)\Common Files\McAfee"
rmdir /s /q "C:\ProgramData\McAfee"
```

Remove the stale Add/Remove Programs registrations found in the source case:
```cmd
reg delete "HKLM\OFFLINE_SOFTWARE\WOW6432Node\Microsoft\Windows\CurrentVersion\Uninstall\MSC" /f
reg delete "HKLM\OFFLINE_SOFTWARE\WOW6432Node\Microsoft\Windows\CurrentVersion\Uninstall\{35ED3F83-4BDC-4c44-8EC6-6A8301C7413A}" /f
```

Remove the remaining McAfee registry trees:
```cmd
reg delete "HKLM\OFFLINE_SOFTWARE\McAfee" /f
reg delete "HKLM\OFFLINE_SOFTWARE\WOW6432Node\McAfee" /f
```

If the source installation has residual kernel-driver registrations that survived the McAfee cleanup, remove the corresponding residual service keys and driver files offline. In the documented case the remaining examples were:
```cmd
reg delete "HKLM\OFFLINE_SYSTEM\ControlSet001\Services\mfencbdc" /f
reg delete "HKLM\OFFLINE_SYSTEM\ControlSet001\Services\mfencrk" /f
del /f C:\Windows\System32\drivers\mfencbdc.sys
del /f C:\Windows\System32\drivers\mfencrk.sys
```

*Do not remove unrelated Windows services, drivers, registry trees, or files.*

Unload the offline hives:
```cmd
reg unload HKLM\OFFLINE_SOFTWARE
reg unload HKLM\OFFLINE_SYSTEM
```

Reboot normally:
```cmd
wpeutil reboot
```

---

### 8. Final verification
After Windows starts, verify that no McAfee services remain:
```powershell
powershell -NoProfile -Command "Get-CimInstance Win32_Service | Where-Object { $_.Name -match 'McAfee|McA|MFE|MMSS' -or $_.DisplayName -match 'McAfee|McA|MFE|MMSS' -or $_.PathName -match 'McAfee|McA|MFE|MMSS' } | Select-Object Name,DisplayName,State,StartMode,PathName"
```

Verify scheduled tasks:
```cmd
schtasks /query /fo LIST 2>nul | findstr /i "McAfee"
```

Verify filesystem filter drivers:
```cmd
fltmc filters | findstr /i "mfe mcafee"
```

Verify driver registrations:
```cmd
sc query type= driver state= all | findstr /i "mfe mcafee"
```

Verify that the main McAfee directories are gone:
```cmd
for %D in ("C:\Program Files\McAfee" "C:\Program Files\Common Files\McAfee" "C:\Program Files (x86)\Common Files\McAfee" "C:\ProgramData\McAfee") do @if exist "%~D" echo STILL EXISTS: %~D
```

Finally verify the registered antivirus:
```powershell
powershell -NoProfile -Command "Get-CimInstance -Namespace root/SecurityCenter2 -ClassName AntiVirusProduct | Select-Object displayName,productState,pathToSignedProductExe"
```

The target result is that no McAfee services/tasks/drivers or McAfee program/data directories remain and Windows Defender is the registered antivirus.
