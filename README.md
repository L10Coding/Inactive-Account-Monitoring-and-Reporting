# 💻⌨️🖥️🖨️ 🖲️🕹️💾🗜️📡📡🧰💎🔧🔨. ⚙️ 🔩🩺🩺🔑🔑🗝️🗝️🖼️🪟📒📘📚📎🖇️📌📍✂️🖊️🖍️📝🔍🔎🔎🔏🔏🔐🔒🔓
Inactive-Account-Monitoring-and-Reporting
Addressing vulnerabilities registering as Inactive/Stale accounts.

# Module Description
Built a PowerShell script that connects to Microsoft Entra ID via Microsoft Graph API, scans every enabled user account in the tenant, checks their last sign-in date, classifies them by inactivity level, and exports a prioritized report of stale accounts that pose a security risk.

**Step 1 - Connect to Microsoft Graph with sign-in audit scope**
- Connected to Entra ID tenant using two scopes
- User.Read.All — permission to read all user accounts
- AuditLog.Read.All — new scope needed to read sign-in activity per user
- Without AuditLog.Read.All the SignInActivity property returns empty for every user
```powershell
Connect-MgGraph -Scopes "User.Read.All","AuditLog.Read.All" -NoWelcome
```
**Step 2 - Pulled all users with sign-in activity**
- Retrieved all 102 user accounts from the tenant
- Added SignInActivity to the property list — this is the new property that holds last sign-in data
- SignInActivity contains: LastSignInDateTime, LastNonInteractiveSignInDateTime, LastSuccessfulSignInDateTime
- Confirmed count with Write-Host
```powershell
$users = Get-MgUser -All -Property "Id,DisplayName,UserPrincipalName,AccountEnabled,SignInActivity"
```
**Step 3 — Built the stale status classification function**
- Takes $LastSignIn as input — the date the user last signed in
- First checks if $LastSignIn is empty using -not — if empty returns "Never Signed In" immediately
- If a date exists subtracts it from today using (Get-Date) - $LastSignIn
- .TotalDays converts the result to a number of days
- [int] drops the decimal to give a clean whole number
- Classifies into four levels: Never Signed In, Inactive 90+, Inactive 60+, Inactive 30+, Active
```powershell
function Get-StaleStatus {
    param($LastSignIn)
    if (-not $LastSignIn)      { return "Never Signed In" }
    $daysSince = [int]((Get-Date) - $LastSignIn).TotalDays
    if ($daysSince -gt 90)     { return "Inactive 90+" }
    elseif ($daysSince -gt 60) { return "Inactive 60+" }
    elseif ($daysSince -gt 30) { return "Inactive 30+" }
    else                       { return "Active" }
}
```
**Step 4 — Built the color coding function**
- Takes a status label and returns a matching console color
- Never signed in and 90+ days both return red — highest risk
- 60+ days returns yellow — moderate risk
- 30+ days returns cyan — low risk
- Active returns green — no action needed
```powershell
function Get-StaleColor {
    param($Status)
    if ($Status -eq "Never Signed In")  { return "Red" }
    elseif ($Status -eq "Inactive 90+") { return "Red" }
    elseif ($Status -eq "Inactive 60+") { return "Yellow" }
    elseif ($Status -eq "Inactive 30+") { return "Cyan" }
    else                                { return "Green" }
}
```
**Step 5 — Created empty results list and captured today's date**
- Created empty list to collect flagged users as the scan runs
- Captured today's date once so every calculation uses the same reference point throughout the script
```powershell
$staleUsers = [System.Collections.Generic.List[PSCustomObject]]::new()
$today      = Get-Date
```
**Step 6 — Built the main scanning loop**
- Looped through all 102 users one at a time
- Skipped disabled accounts immediately using continue
- Grabbed LastSignInDateTime from the SignInActivity property on each user
- Called Get-StaleStatus to classify the user based on days since last sign-in
- Called Get-StaleColor to get the matching color for that status
- Skipped active users using continue — only flagged users get printed and saved
- For flagged users printed a color coded line showing name, UPN, last sign-in date and status
- Added each flagged user to $staleUsers as a custom object with five properties
- Used inline if on LastSignInDate — if date exists format it as yyyy-MM-dd, if null save as "Never"
```powershell
foreach ($user in $users) {
    if ($user.AccountEnabled -eq $false) { continue }
    $lastSignIn = $user.SignInActivity.LastSignInDateTime
    $status     = Get-StaleStatus -LastSignIn $lastSignIn
    $color      = Get-StaleColor -Status $status
    if ($status -eq "Active") { continue }
    Write-Host "User: $($user.DisplayName) | UPN: $($user.UserPrincipalName) | Last Sign In: $lastSignIn | Status: $status" -ForegroundColor $color
    $staleUsers.Add([PSCustomObject]@{ ... })
}
```
**Step 7 — Exported stale users to CSV**
- Exported $staleUsers list to StaleUsers_Report.csv on Desktop
- Sorted by Status so users are grouped by inactivity level in the report
- -NoTypeInformation removes the ugly header line PowerShell adds by default
- Report includes: DisplayName, UserPrincipalName, AccountEnabled, LastSignInDate, Status
```powershell
$staleUsers | Sort-Object Status | Export-Csv -Path "$HOME/Desktop/StaleUsers_Report.csv" -NoTypeInformation
```
**Step 8 — Calculated counts per inactivity level**
- Filtered $staleUsers by each status level using Where-Object
- () wraps each filter so PowerShell runs the filter first then counts the result
- .Count gives the number of users in each category
- Four separate counts used in the summary box below
```powershell
$neverSignedIn = ($staleUsers | Where-Object Status -eq "Never Signed In").Count
$inactive90    = ($staleUsers | Where-Object Status -eq "Inactive 90+").Count
$inactive60    = ($staleUsers | Where-Object Status -eq "Inactive 60+").Count
$inactive30    = ($staleUsers | Where-Object Status -eq "Inactive 30+").Count
```
**Step 9 — Printed formatted summary**
- Printed a clean formatted summary box
- Total users scanned in white
- Stale users and never signed in in red — highest risk
- 60+ days in yellow — moderate risk
- 30+ days in cyan — lower risk
```powershell
Write-Host "  STALE USER SCAN SUMMARY"
Write-Host "  Total users scanned : $($users.Count)"
Write-Host "  Stale users found   : $($staleUsers.Count)"
Write-Host "  Never signed in     : $neverSignedIn"
Write-Host "  Inactive 90+ days   : $inactive90"
Write-Host "  Inactive 60+ days   : $inactive60"
Write-Host "  Inactive 30+ days   : $inactive30"
```
**Results from PowerShell summary screen**

<img width="579" height="325" alt="Screenshot 2026-07-28 at 22 23 12" src="https://github.com/user-attachments/assets/c2b2a62e-cbe3-4eb3-9481-ab0f97d33baf" />

**Results from CSV file**

<img width="914" height="854" alt="Screenshot 2026-07-28 at 22 23 51" src="https://github.com/user-attachments/assets/8d98a243-7905-4e9c-9ee7-7a4d3c2ac1e7" />
