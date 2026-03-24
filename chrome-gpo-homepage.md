# Chrome Homepage via GPO — Set Google Domain-Wide

Quick runbook for pushing Google as the locked homepage in Google Chrome across all domain-joined machines. Uses a Global GPO at the domain root so it applies to everyone.

---

## What you need

- Windows Server 2025 DC
- Domain Admin credentials
- RDP access to the DC
- Chrome ADMX templates (grabbed in Step 1)

---

## Step 1 — Grab the Chrome ADMX Templates

Same deal as Edge — Chrome doesn't ship with ADMX templates in Windows Server. You have to add them.

1. Hit up: [https://chromeenterprise.google/browser/download/](https://chromeenterprise.google/browser/download/)
2. Scroll to **"Chrome Browser policy templates"** → download the **ADM/ADMX templates (.zip)**
3. Extract it — you need both of these:

```
googlechromeadmx\
  └── windows\
        └── admx\
              ├── chrome.admx
              ├── google.admx        ← don't skip this one
              └── en-US\
                    ├── chrome.adml
                    └── google.adml
```

> Both `chrome.admx` **and** `google.admx` are required. Missing `google.admx` will break the template load.

---

## Step 2 — Copy the Files to the Central Store

> Don't try to copy over the network share — you'll get access denied. Just RDP into the DC and run this directly.

Open **PowerShell as Administrator** on the DC:

```powershell
# Create the Central Store if it doesn't exist yet
New-Item -Path "C:\Windows\SYSVOL\sysvol\corp.local\Policies\PolicyDefinitions" -ItemType Directory -Force
New-Item -Path "C:\Windows\SYSVOL\sysvol\corp.local\Policies\PolicyDefinitions\en-US" -ItemType Directory -Force

# Copy the ADMX files (adjust source path to wherever you extracted the ZIP)
Copy-Item "C:\Temp\chrome.admx" -Destination "C:\Windows\SYSVOL\sysvol\corp.local\Policies\PolicyDefinitions\"
Copy-Item "C:\Temp\google.admx" -Destination "C:\Windows\SYSVOL\sysvol\corp.local\Policies\PolicyDefinitions\"

Copy-Item "C:\Temp\en-US\chrome.adml" -Destination "C:\Windows\SYSVOL\sysvol\corp.local\Policies\PolicyDefinitions\en-US\"
Copy-Item "C:\Temp\en-US\google.adml" -Destination "C:\Windows\SYSVOL\sysvol\corp.local\Policies\PolicyDefinitions\en-US\"
```

Double-check they landed:

```powershell
Get-ChildItem "C:\Windows\SYSVOL\sysvol\corp.local\Policies\PolicyDefinitions\" | Where-Object {$_.Name -like "*chrome*" -or $_.Name -like "*google*"}
```

You should see:
```
chrome.admx
google.admx
```

---

## Step 3 — Create the GPO

1. Open `gpmc.msc`
2. Expand **Forest → Domains → corp.local**
3. Right-click the domain → **Create a GPO in this domain, and Link it here**
4. Name it `Global - Chrome Homepage` → **OK**
5. Right-click the new GPO → **Edit**

---

## Step 4 — Configure the Policies

Close and reopen the editor after copying the ADMX files or Google Chrome won't show up in the list.

Navigate to:

```
Computer Configuration
  └── Policies
        └── Administrative Templates
              └── Google
                    └── Google Chrome
                          └── Startup, Home page and New Tab page
```

| Policy | State | Value |
|---|---|---|
| Action on startup | Enabled | Open a specific page or set of pages |
| URLs to open on startup | Enabled | `https://www.google.com` |
| Configure the home page URL | Enabled | `https://www.google.com` |
| Show Home button on toolbar | Enabled | — |
| Lock home page to | Enabled | — |
| HomepageIsNewTabPage | Disabled | — |

---

## Step 5 — Enforce It

Right-click the **Global - Chrome Homepage** link under your domain in GPMC → **Enforced**

This locks it down so child OUs with Block Inheritance can't dodge the policy.

---

## Step 6 — Verify

```powershell
# Push the update
gpupdate /force

# Check that the policy landed
reg query "HKLM\SOFTWARE\Policies\Google\Chrome"

# Full GPO report
gpresult /r /scope computer
```

Look for `Global - Chrome Homepage` under **Applied Group Policy Objects**.

---

## Troubleshooting

| Problem | Fix |
|---|---|
| Google Chrome missing from Administrative Templates | Make sure both `chrome.admx` AND `google.admx` are in the Central Store, then close/reopen GPMC |
| Access denied copying to SYSVOL | RDP into the DC and copy locally as Domain Admin |
| Policy not hitting some machines | Check for Block Inheritance on child OUs — make sure the GPO is set to Enforced |
| reg query comes back empty | Run `gpupdate /force`, confirm the machine is domain-joined |
