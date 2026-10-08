# 1️⃣0️⃣ Windows Activation Transfer

**OLD LAPTOP**
Open **Command Prompt as Administrator**: `slmgr /dlv`
If it's an OEM key stored in the laptop's firmware, run: `wmic path softwarelicensingservice get OA3OriginalProductKey`
You can also use PowerShell: `(Get-CimInstance -ClassName SoftwareLicensingService).OA3OriginalProductKey`
Open **Command Prompt as Administrator**: `slmgr /upk`
Then: `slmgr /cpky`
This removes the installed product key from the old Windows installation.

**NEW LAPTOP**
Then open **Command Prompt as Administrator**: `slmgr /ipk XXXXX-XXXXX-XXXXX-XXXXX-XXXXX`
Then activate: `slmgr /ato`
Check activation: `slmgr /xpr`
