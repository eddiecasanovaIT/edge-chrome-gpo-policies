# Edge Homepage via GPO — Set Google Domain-Wide

Quick runbook for pushing Google as the locked homepage in Microsoft Edge across all domain-joined machines. Uses a Global GPO at the domain root so it applies to everyone.

---

## What you need

- Windows Server 2025 DC
- Domain Admin credentials
- RDP access to the DC
- Edge ADMX templates (grabbed in Step 1)

---

## Step 1 — Grab the Edge ADMX Templates

Edge doesn't come with ADMX templates baked into Windows Server — you have to add them yourself.

1. Hit up: [https://www.microsoft.com/en-us/edge/business/download](https://www.microsoft.com/en-us/edge/business/download)
2. Scroll to **"Policy files"** → download the **Policy files (.zip)** for your Edge version
3. Extract it — you're looking for these files:

```
MicrosoftEdgePolicyTemplates\
  └── windows\
        └── admx\
              ├── msedge.admx
              ├── msedgeupdate.admx
              ├── msedgewebview2.admx
              └── en-US\
                    ├── msedge.adml
                    ├── msedgeupdate.adml
                    └── msedgewebview2.adml
```

---

## Step 2 — Copy the Files to the Central Store

> Don't try to copy over the network share — you'll get access denied. Just RDP into the DC and run this directly.

Open **PowerShell as Administrator** on the DC:

```powershell
# Create the Central Store if it doesn't exist yet
New-Item -Path "C:\Windows\SYSVOL\sysvol\corp.local\Policies\PolicyDefinitions" -ItemType Directory -Force
New-Item -Path "C:\Windows\SYSVOL\sysvol\corp.local\Policies\PolicyDefinitions\en-US" -ItemType Directory -Force

# Copy the ADMX files (adjust the source path to wherever you extracted the ZIP)
Copy-Item "C:\Temp\msedge.admx" -Destination "C:\Windows\SYSVOL\sysvol\corp.local\Policies\PolicyDefinitions\"
Copy-Item "C:\Temp\msedgeupdate.admx" -Destination "C:\Windows\SYSVOL\sysvol\corp.local\Policies\PolicyDefinitions\"
Copy-Item "C:\Temp\msedgewebview2.admx" -Destination "C:\Windows\SYSVOL\sysvol\corp.local\Policies\PolicyDefinitions\"

Copy-Item "C:\Temp\en-US\msedge.adml" -Destination "C:\Windows\SYSVOL\sysvol\corp.local\Policies\PolicyDefinitions\en-US\"
Copy-Item "C:\Temp\en-US\msedgeupdate.adml" -Destination "C:\Windows\SYSVOL\sysvol\corp.local\Policies\PolicyDefinitions\en-US\"
Copy-Item "C:\Temp\en-US\msedgewebview2.adml" -Destination "C:\Windows\SYSVOL\sysvol\corp.local\Policies\PolicyDefinitions\en-US\"
```

Double-check they landed:

```powershell
Get-ChildItem "C:\Windows\SYSVOL\sysvol\corp.local\Policies\PolicyDefinitions\" | Where-Object {$_.Name -like "*edge*"}
```

You should see:
```
msedge.admx
msedgeupdate.admx
msedgewebview2.admx
```

---

## Step 3 — Create the GPO

1. Open `gpmc.msc`
2. Expand **Forest → Domains → corp.local**
3. Right-click the domain → **Create a GPO in this domain, and Link it here**
4. Name it `Global - Edge Homepage` → **OK**
5. Right-click the new GPO → **Edit**

---

## Step 4 — Configure the Policies

Close and reopen the editor after copying the ADMX files or Microsoft Edge won't show up in the list.

Navigate to:

```
Computer Configuration
  └── Policies
        └── Administrative Templates
              └── Microsoft Edge
                    └── Startup, home page and new tab page
```

| Policy | State | Value |
|---|---|---|
| Configure the home page URL | Enabled | `https://www.google.com` |
| Show Home button on toolbar | Enabled | — |
| Lock home page | Enabled | — |
| HomepageIsNewTabPage | Disabled | — |

---

## Step 5 — Enforce It

Right-click the **Global - Edge Homepage** link under your domain in GPMC → **Enforced**

This locks it down so child OUs with Block Inheritance can't dodge the policy.

---

## Step 6 — Verify

```powershell
# Push the update
gpupdate /force

# Check that the policy landed
reg query "HKLM\SOFTWARE\Policies\Microsoft\Edge"

# Full GPO report
gpresult /r /scope computer
```

Look for `Global - Edge Homepage` under **Applied Group Policy Objects**.

---

## Troubleshooting

| Problem | Fix |
|---|---|
| Microsoft Edge missing from Administrative Templates | ADMX files didn't copy correctly — recheck the path and close/reopen GPMC |
| Access denied copying to SYSVOL | RDP into the DC and copy locally as Domain Admin |
| Policy not hitting some machines | Check for Block Inheritance on child OUs — make sure the GPO is set to Enforced |
| reg query comes back empty | Run `gpupdate /force`, confirm the machine is domain-joined |
