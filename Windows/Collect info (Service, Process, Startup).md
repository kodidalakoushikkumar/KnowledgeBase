# 9️⃣ Collect info (Service, Process, Startup)

**Windows system enumeration**. Collect information about:

1. **Running processes**
2. **Whether process executables are digitally signed**
3. **Programs configured to start automatically**
4. **Currently running services**

This is useful for **troubleshooting, malware investigation, incident response, and general system auditing**.

```
Get-CimInstance Win32_Process |
Select-Object Name, ProcessId, ParentProcessId, ExecutablePath, CommandLine |
Format-List |
Out-File "$env:USERPROFILE\Desktop\processes.txt" -Width 300
```

```
Get-CimInstance Win32_Process |
Where-Object { $_.ExecutablePath } |
ForEach-Object {
    $sig = Get-AuthenticodeSignature $_.ExecutablePath

    [PSCustomObject]@{
        Process = $_.Name
        PID     = $_.ProcessId
        Path    = $_.ExecutablePath
        Status  = $sig.Status
        Signer  = $sig.SignerCertificate.Subject
    }
} |
Out-File "$env:USERPROFILE\Desktop\process_signatures.txt" -Width 300
```

```
Get-CimInstance Win32_StartupCommand |
Select-Object Name, Command, Location, User |
Format-List |
Out-File "$env:USERPROFILE\Desktop\startup.txt" -Width 300
```

```
Get-CimInstance Win32_Service |
Where-Object { $_.State -eq "Running" } |
Select-Object Name, DisplayName, State, StartMode, PathName |
Format-List |
Out-File "$env:USERPROFILE\Desktop\services.txt" -Width 300
```






