

# 5️⃣ Bitlocker

### **Prerequisites and Verification**

Before starting, ensure your system has **TPM 2.0 enabled**, is in **UEFI Boot Mode**, and you have **Local Administrator privileges**.

1. **Verify TPM Status:**
    - Press `Windows + R`, type **tpm.msc**, and press Enter.
    - Confirm the status is "The TPM is ready for use" and the version is 2.0.
2. **Verify UEFI Mode:**
    - Press `Windows + R`, type **msinfo32**, and press Enter.
    - Confirm "BIOS Mode" is set to **UEFI**.
3. **Check Current BitLocker Status:**
    - Open **Command Prompt as Administrator** and run: `manage-bde -status`
    - The "Protection Status" should be **Off** before you begin.

### **Configuration Steps**

1. **Create a Recovery Key:**
    - Open **PowerShell as Administrator** and run: `Add-BitLockerKeyProtector -MountPoint "C:" -RecoveryPasswordProtector`
    - **Crucial:** Save the 48-digit recovery key provided in a secure location.
    - Verify it was added by running: `manage-bde -protectors -get C:`.
2. **Configure Group Policy for PIN:**
    - Run **gpedit.msc** and navigate to: `Computer Configuration > Administrative Templates > Windows Components > BitLocker Drive Encryption > Operating System Drives`.
    - Open **"Require additional authentication at startup"**, set it to **Enabled**, and select **"Require startup PIN with TPM"**.
    - Apply the changes and force a policy update by running: **gpupdate /force**.
3. **Add the TPM + PIN Protector:**
    - In **Command Prompt (Admin)**, run: `manage-bde -protectors -add C: -TPMAndPIN`
    - Enter and confirm your chosen PIN (an 8-digit numeric PIN is recommended).
4. **Enable BitLocker Encryption:**
    - Run the following command to start encryption (using "used space only" for speed): `manage-bde -on C: -usedspaceonly`.
5. **Reboot and Validate:**
    - Restart the laptop: `shutdown /r /t 0`.
    - The BitLocker pre-boot screen should appear; enter your PIN to boot into Windows.
6. **Final Verification:**
    - Once logged in, check the status: `manage-bde -status C:`
    - Success is confirmed if "Percentage Encrypted" is **100%** and "Protection Status" is **On**.

### **Additional Commands**

- **Encrypt a Secondary Drive (D:):**
    - `manage-bde -on D: -usedspaceonly`
    - `manage-bde -protectors -add D: -RecoveryPassword`.
- **Change your BitLocker PIN:**
    - `manage-bde -changepin C:`.
