# edge-chrome-gpo-policies

GPO configs for locking down Microsoft Edge and Google Chrome behavior across a Windows AD domain. No fluff — just the steps that work.

---

## What's in here

Runbooks for managing browser settings via Group Policy on **Windows Server 2025**. Everything is deployed through the Central Store using Computer Configuration so it hits every machine on the domain regardless of who's logged in.

| Guide | Browser | What it does |
|---|---|---|
| [Edge - Set Google as Homepage](./edge-gpo-homepage.md) | Microsoft Edge | Pushes and locks Google as the homepage domain-wide |
| [Chrome - Set Google as Homepage](./chrome-gpo-homepage.md) | Google Chrome | Pushes and locks Google as the homepage domain-wide |

---

## Repo Structure

```
edge-chrome-gpo-policies/
├── README.md
├── edge-gpo-homepage.md
└── chrome-gpo-homepage.md
```

---

## Environment

| | |
|---|---|
| Domain | corp.local |
| Server OS | Windows Server 2025 |
| GPO Scope | Domain root (Global) |
| Config Target | Computer Configuration |

---

## How these guides are structured

Every runbook follows the same pattern:

1. Download the browser's official ADMX templates
2. Drop them into the **Central Store** on the DC via PowerShell (remote copy will get you an access denied — just RDP in directly)
3. Create a **Global GPO** linked at the domain root
4. Configure settings under **Computer Configuration → Administrative Templates**
5. Set it to **Enforced** so child OUs can't override it
6. Verify with `gpupdate /force` and `gpresult /r`

---

## Notes

- Neither Edge nor Chrome ships with ADMX templates built into Windows Server — you have to add them manually. Each guide covers this.
- Both sets of ADMX files live in the same Central Store without conflicting.
- More browser policy guides (extension management, security settings, etc.) coming over time.
