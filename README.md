
# ==========================================
# OneDrive / SharePoint Sync Diagnostic Tool
# ==========================================

# REPORT
$Report = "c:\temp\OneDrive_Full_Diagnostic_Report.txt"
"===== OneDrive Full Diagnostic Report =====" | Out-File $Report
"Generated: $(Get-Date)" | Out-File -Append $Report

# ==========================================
# 1. INSTALLATION TYPE
# ==========================================
"`n--- OneDrive Installation ---" | Out-File -Append $Report

$MachineInstall = Test-Path "C:\Program Files\Microsoft OneDrive\OneDrive.exe"
$UserInstall = Test-Path "$env:LOCALAPPDATA\Microsoft\OneDrive\OneDrive.exe"

if ($MachineInstall) {
    "✔ Machine-wide installation detected" | Out-File -Append $Report
}
elseif ($UserInstall) {
    "✔ Per-user installation detected" | Out-File -Append $Report
}
else {
    "❌ OneDrive not installed" | Out-File -Append $Report
}

# ==========================================
# 2. PROCESS CHECK
# ==========================================
"`n--- Running Instances ---" | Out-File -Append $Report

$OD = Get-Process OneDrive -ErrorAction SilentlyContinue

if ($OD) {
    "✔ OneDrive is running ($($OD.Count) instance(s))" | Out-File -Append $Report
} else {
    "❌ OneDrive not running" | Out-File -Append $Report
}

# ==========================================
# 3. ACCOUNT DETECTION
# ==========================================
"`n--- Configured Accounts ---" | Out-File -Append $Report

$Accounts = Get-ItemProperty "HKCU:\Software\Microsoft\OneDrive\Accounts\*" -ErrorAction SilentlyContinue

if (!$Accounts) {
    "❌ No OneDrive accounts configured" | Out-File -Append $Report
    exit
}

foreach ($Acc in $Accounts) {

    "Account: $($Acc.DisplayName)" | Out-File -Append $Report
    "Email  : $($Acc.UserEmail)" | Out-File -Append $Report
    "Folder : $($Acc.UserFolder)" | Out-File -Append $Report

    if (!(Test-Path $Acc.UserFolder)) {
        "⚠️ Reason: Sync folder missing → Fix: Re-link OneDrive" | Out-File -Append $Report
        continue
    }

    $RootPath = $Acc.UserFolder

    # ==========================================
    # 4. FILE SCAN
    # ==========================================
    "`n--- File Scan ($RootPath) ---" | Out-File -Append $Report

    $MaxPath = 240
    $InvalidChars = '[\"*:<>?/\\|]'
    $Reserved = @("CON","PRN","AUX","NUL","COM1","COM2","COM3","COM4","COM5","COM6","COM7","COM8","COM9","LPT1","LPT2","LPT3","LPT4","LPT5","LPT6","LPT7","LPT8","LPT9")

    $IssueCount = 0
    $ItemCount = 0

    $Files = Get-ChildItem -LiteralPath $RootPath -Recurse -Force -ErrorAction SilentlyContinue

    foreach ($File in $Files) {

        $ItemCount++
        $Issues = @()

        $Name = $File.Name
        $FullPath = $File.FullName

        # Long path
        if ($FullPath.Length -ge $MaxPath) {
            $Issues += "LongPath → Reason: Exceeds 240 characters → Fix: Shorten path"
        }

        # Invalid characters
        if ($Name -match $InvalidChars) {
            $Issues += "InvalidChar → Reason: Unsupported characters → Fix: Rename"
        }

        # Leading/trailing space or dot
        if ($Name -match '(^\s)|(\s$)|(\.$)') {
            $Issues += "NamingIssue → Reason: Leading/trailing space or dot → Fix: Rename"
        }

        # Reserved names
        $Base = [System.IO.Path]::GetFileNameWithoutExtension($Name)
        if ($Reserved -contains $Base.ToUpper()) {
            $Issues += "ReservedName → Reason: Windows reserved name → Fix: Rename"
        }

        # System attribute
        if ($File.Attributes -match "System") {
            $Issues += "SystemFile → Reason: System attribute may block sync → Fix: attrib -s"
        }

        # Temp files
        if ($Name -like "~$*") {
            $Issues += "TempFile → Reason: Temporary Office file → Fix: Close app/delete"
        }

        # Metadata files
        if ($File.Extension -in ".ini",".db") {
            $Issues += "Metadata → Reason: App/system file → Fix: Move outside OneDrive"
        }

        # Control characters (NOT normal Unicode)
        if ($Name -match '[\x00-\x1F]') {
            $Issues += "InvalidUnicode → Reason: Control characters → Fix: Rename"
        }

        if ($Issues.Count -gt 0) {
            $IssueCount++
            "$FullPath --> $($Issues -join ' | ')" | Out-File -Append $Report
        }
    }

    "Scanned: $ItemCount items" | Out-File -Append $Report
    "Issues : $IssueCount found" | Out-File -Append $Report
}

# ==========================================
# 5. POLICY CHECK
# ==========================================
"`n--- Policy Check ---" | Out-File -Append $Report

$Policies = Get-ItemProperty "HKLM:\SOFTWARE\Policies\Microsoft\OneDrive" -ErrorAction SilentlyContinue

if ($Policies) {
    if ($Policies.DisableFileSyncNGSC -eq 1) {
        "❌ Sync disabled by policy → Fix: Disable GPO" | Out-File -Append $Report
    }

    if ($Policies.BlockedFileExtensions) {
        "⚠️ Blocked file types: $($Policies.BlockedFileExtensions)" | Out-File -Append $Report
    }
} else {
    "✔ No blocking policies detected" | Out-File -Append $Report
}

# ==========================================
# 6. LOG ANALYSIS
# ==========================================
"`n--- Log Analysis ---" | Out-File -Append $Report

$LogPath = "$env:LOCALAPPDATA\Microsoft\OneDrive\logs"
$Patterns = @("0x8004","UploadBlocked","FileLocked","AccessDenied")

if (Test-Path $LogPath) {
    $Errors = Get-ChildItem "$LogPath\*.log" -ErrorAction SilentlyContinue |
        Select-String -Pattern $Patterns |
        Select-Object -Last 10

    foreach ($E in $Errors) {

        $Line = $E.Line
        $Reason = "Unknown"
        $Fix = "Check logs"

        if ($Line -match "0x8004") { $Reason = "Auth issue"; $Fix = "Re-login" }
        elseif ($Line -match "UploadBlocked") { $Reason = "Policy block"; $Fix = "Check permissions" }
        elseif ($Line -match "FileLocked") { $Reason = "File in use"; $Fix = "Close application" }
        elseif ($Line -match "AccessDenied") { $Reason = "Permission issue"; $Fix = "Verify access" }

        "$Line → Reason: $Reason → Fix: $Fix" | Out-File -Append $Report
    }
}

# ==========================================
# FINAL
# ==========================================
Write-Host "✅ Diagnostic complete. Report saved to Desktop." -ForegroundColor Green
