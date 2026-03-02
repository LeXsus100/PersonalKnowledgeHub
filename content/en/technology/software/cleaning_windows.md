---
title: "Cleaning & Boost Windows"
description: ""
summary: ""
date: 2023-09-07T16:04:48+02:00
lastmod: 2023-09-07T16:04:48+02:00
draft: false
weight: 19
toc: true
seo:
  title: "" # custom title (optional)
  description: "" # custom description (recommended)
  canonical: "" # custom canonical URL (optional)
  noindex: false # false (default) or true
---
This document provides a **structured, safe, and progressive workflow** to clean, verify, and optimize a Windows system.  
Follow the steps **in order**.  
*Optional* and *situational* actions are clearly marked at the end.

---

## 1. Preparation and Safety

Before making any system-level changes, ensure rollback capability and baseline stability.

**Create a System Restore Point**  
Open *Advanced System Settings* → *System Protection* → *Create*.  
This allows recovery in case of misconfiguration.

**Run the built-in Windows Troubleshooter**  
Use the general troubleshooting tools to automatically detect common issues.

**Review System Settings**  
Perform a quick audit of Windows settings (privacy, updates, background apps) to identify obvious misconfigurations.

{{< callout context="note" title="Note" icon="outline/info-circle" >}}
Skipping this phase increases the risk of irreversible issues later.
{{< /callout >}}

---

## 2. Startup

Reduce unnecessary load and eliminate unused components.

**Disable Startup Applications**  
Open *Task Manager* → *Startup* tab → disable non-essential programs.

**Uninstall Unused Software**  
Remove applications you no longer use.  
Optionally perform a general cleanup using tools such as *BleachBit*.

**Disk Maintenance**  
- If using an **HDD**, run disk defragmentation.  
- Perform a disk error check via *Drive Properties*.

{{< callout context="caution" title="Caution" icon="outline/alert-triangle" >}}
Do **not** defragment SSDs. Windows manages SSD optimization automatically.
{{< /callout >}}

---

## 3. Drivers and Updates

Ensure the system is current, stable, and not overburdened by unnecessary services.

**Apply Recommended Privacy Tweaks**  
Install *WPD* and enable the **recommended** settings.

**Update Drivers**  
Check manufacturer drivers manually or use *SDIO* for detection assistance.

**Install Windows Updates**  
Ensure Windows is fully up to date before proceeding further.

---

## 4. Security and Malwares

Confirm system integrity before optimization.

**Antivirus and Anti-malware Scan**  
Run a full antivirus and a malware scan (free trials are sub-optimal).  

{{< link-card title="List of Security Software for Windows" href="/technology/software/software_windows/#security--privacy" >}}

{{< callout context="danger" title="Danger" icon="outline/alert-octagon" >}}
Do not optimize or debloat a system that may still be infected.
{{< /callout >}}

---

## 5. System Integrity Checks

Run the following commands **as Administrator**, in this order:

1. **Disk Health Check**
   
   ```bash
   wmic diskdrive get status
   ```

2. **System File Checker**

   ```bash
   sfc /scannow
   ```

3. **DISM Health Scan and Repair**

   In sequence:

   ```bash
   DISM /Online /Cleanup-Image /ScanHealth
   DISM /Online /Cleanup-Image /RestoreHealth
   ```

4. **Disk Repair (Requires Reboot)**

   ```bash
   chkdsk /f /r /x
   ```

**Memory Test**  
Run *Windows Memory Diagnostic*.  
This requires a reboot and may take significant time.

---

## 6. Software Update Verification

Ensure third-party software is current.

**PowerShell (Admin)**

```powershell
winget upgrade --all
```

{{< callout context="tip" title="Did you know?" icon="outline/rocket" >}}
`winget` allows centralized, scriptable package management similar to Linux systems.
{{< /callout >}}

---

## 7. Performance Boosting

Only proceed once the system is confirmed stable.

**Debloating Script**  
Run in *PowerShell (Administrator)* and follow the instructions:

```powershell
& ([scriptblock]::Create((irm "https://debloat.raphi.re/")))
```

For more information on this Windows11 debloater follow the [github project](https://github.com/Raphire/Win11Debloat).

**Disable Visual Animations**  
Advanced System Settings → Performance Options → disable animations.
Keep **“Smooth edges of screen fonts”** enabled.

**Battery and Power Configuration (Laptops)**  
Review battery usage and power plans.
Power Options → Advanced settings → verify CPU and power limits.

---

## 8. Optional and Situational

**Service Optimization**  
`msconfig` → Services → *Hide all Microsoft services* → disable non-essential services only.

**Boot Optimization**  
`msconfig` → Boot:

* Disable GUI boot
* Set timeout to 5–10 seconds
* Advanced options → set maximum number of processors

**Disk Indexing (*Situational*)**  
Drive Properties → uncheck *Allow files on this drive to have contents indexed*.

**Audio Optimization (*Situational*)**  
Sound → Speakers → Properties → Advanced → set minimum format (16-bit).

{{< callout context="caution" title="Caution" icon="outline/alert-triangle" >}}
Aggressive service or boot optimizations may reduce compatibility or stability.
{{< /callout >}}

---

## Final Notes

* Always prioritize **stability over marginal performance gains**.
* Apply changes incrementally and test between steps.
* Maintain regular backups and restore points.