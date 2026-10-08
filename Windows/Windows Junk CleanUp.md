
# 7️⃣ Windows Junk CleanUp

If `C:\Windows\WinSxS` and `C:\Windows\SoftwareDistribution\Download` together are using around **100 GB**, that's much larger than expected in most cases. The good news is that you **shouldn't manually delete WinSxS contents**, but there are safe ways to reclaim space.
### 1. Check the actual size of WinSxS \(recommended first\)

Windows Explorer often **overstates** the size of `WinSxS` because of hard links.
Open **Command Prompt as Administrator** and run: `Dism.exe /Online /Cleanup-Image /AnalyzeComponentStore`
This tells you:

- Actual size
- Whether cleanup is recommended
- Potential reclaimable space
### 2. Clean up WinSxS safely

If cleanup is recommended: `Dism.exe /Online /Cleanup-Image /StartComponentCleanup`
For a more aggressive cleanup \(prevents uninstalling old updates\): `Dism.exe /Online /Cleanup-Image /StartComponentCleanup /ResetBase`
**Use **`/ResetBase`** only if you're comfortable giving up the ability to uninstall currently installed Windows updates.**
### 3. Clean `SoftwareDistribution\Download`

This folder stores downloaded Windows Update files. You can safely clear it.
Method 1 \(recommended\)
Open Command Prompt as Administrator:

```
net stop wuauserv
net stop bits
```

Then delete everything inside: `C:\Windows\SoftwareDistribution\Download` , Do **not** delete the `SoftwareDistribution` folder itself.
Then restart services:

```
net start bits
net start wuauserv
```

Windows will recreate anything it needs.
### 4. Run Disk Cleanup

Press **Win + R** RUN: `cleanmgr`
Choose **C:**
Click **Clean up system files**.
Check:

- Windows Update Cleanup
- Delivery Optimization Files
- Temporary Windows installation files
- Temporary files
- Previous Windows installation \(Windows.old\), if present

This often frees several GB.
### 5. Check for Windows.old

Look for: `C:\Windows.old`
This alone can consume **20–40 GB**.
Remove it through Disk Cleanup rather than deleting it manually.
### 6. Check if updates are stuck

If `SoftwareDistribution\Download` alone is tens of GB, Windows Update may have repeatedly downloaded failed updates.
Run: `Get-WindowsUpdateLog`
or check: `Settings → Windows Update` for failed updates.

If WinSxS is still huge \(50–80 GB\)
Run: `Dism /Online /Cleanup-Image /RestoreHealth`
followed by: `sfc /scannow`