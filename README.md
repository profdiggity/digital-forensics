# iOS Geolocation Forensics: Backup, Artifact Discovery, &amp; Analysis

## Overview

This guide outlines the process for performing a secure backup of an iOS device, identifying and extracting geolocation artifacts, and analyzing those artifacts using freely-available tools and reproducible methods. The context is a forensic investigation in which a specific iPhone device is being examined to determine location history and movement patterns. We advise against relying on cloud-only methods and focus instead on secure, local, encrypted backups and artifact-level analysis using cross-platform tools compatible with Debian-based operating systems.

## 🔐 Step 1: Create a Secure, Encrypted Backup

A local, **encrypted** backup is required to access sensitive artifacts like **Significant Locations**, **Health data**, and **Apple Maps history**. Unencrypted backups will omit this data.

### Option A — Backup Using Finder (macOS Catalina and later)

1. Connect the iPhone via Lightning cable (or USB-C for later models).
2. Open **Finder** and select the device under "Locations."
3. Under "Backups," choose:
   - "Back up all of the data on your iPhone to this Mac"
   - Check "Encrypt local backup" and set a memorable password.
4. Click **Back Up Now**.
5. Confirm completion by checking the latest backup time.

### Option B — Backup Using iTunes (macOS Mojave or Windows)

1. Open iTunes and select the device icon when detected.
2. In the "Summary" tab:
   - Choose "Back up to this computer"
   - Check "Encrypt iPhone backup"
3. Click **Back Up Now**.
4. Confirm the backup via iTunes preferences > Devices tab.

### Option C — Verify iCloud Backup (if applicable)

If local backup isn't feasible, verify iCloud status:

1. On iPhone: Settings → [Your Name] → iCloud → iCloud Backup
2. Confirm it is ON.
3. Tap **Back Up Now** while connected to Wi-Fi and charging.
> ⚠️ Note: iCloud backup must have enough space and will not include all forensic artifacts. Prioritize local, encrypted backups whenever possible.

## 📁 Step 2: Locate the iOS Backup on Disk (macOS or Windows)

Before you can explore the backup with third-party tools or manually extract geolocation databases, you need to find where the backup is stored on your system.

### 🍎 On macOS (Catalina and newer):

1. Open **Finder**
2. From the menu bar: `Go → Go to Folder…`
3. Enter:
```bash
~/Library/Application Support/MobileSync/Backup/
```
4. Inside, you'll find one or more folders with long hexadecimal names: these are your backups. The newest one is usually the last modified.

Alternatively:
- Finder → Select iPhone → **Manage Backups** → Right-click backup → **Show in Finder**

### ⌛ On macOS (Mojave or earlier) or Windows with iTunes:

- macOS: Same path as above
- Windows:
```bash
%APPDATA%\Apple Computer\MobileSync\Backup\
```
Paste that into File Explorer.

> Backup folders use hashed filenames. Tools like iPhone Backup Extractor or iMazing can decode and extract readable files.

## 🔓 Step 3: Extract &amp; Explore Backup Contents

### 🗃️ iPhone Backup Extractor (Reincubate)

- Download: [https://reincubate.com/iphone-backup-extractor](https://reincubate.com/iphone-backup-extractor/)
- Open the app. It auto-detects backups
- Select a backup and explore by category (Messages, Locations, etc.)
- Export as CSV, PDF, or media

> Supports encrypted backups and can extract SQLite files, PLISTs, and raw data structures.

### 📦 iMazing (Paid)

- iMazing is another option but requires a commercial license.
- Download: [https://imazing.com](https://imazing.com)

## 📍 Step 4: Identify Geolocation Artifacts on iOS

iOS stores location history in structured formats including SQLite databases and PLISTs.

### 📖 Routined (Apple's Learned Location System)
```bash
/private/var/mobile/Library/Caches/com.apple.routined/Cache.sqlite
/private/var/mobile/Library/Caches/com.apple.routined/Local.sqlite
```

Contains learned visits, coordinates, durations, and confidence scores.

### 🗺️ Apple Maps Artifacts
```bash
/private/var/mobile/Containers/Data/Application/[APPGUID]/Library/Maps/GeoHistory.mapsdata
/private/var/mobile/Containers/Data/Application/[APPGUID]/Library/Maps/GeoBookmarks.plist
```

Contains Apple Maps history and saved locations.

> Requires full filesystem access or decryption from a valid encrypted backup.

## 🔬 Step 5: Analyze Geolocation Data Using Open Source Tools

### Recommended: `sqlitebrowser` (DB Browser for SQLite)

Install on Debian-based systems:
```bash
sudo apt install sqlitebrowser
```

### To Use:
1. Launch `sqlitebrowser`
2. Open `Cache.sqlite` or `Local.sqlite`
3. Browse tables:
   - `ZRTLEARNEDVISITMO`
   - `ZRTLEARNEDLOCATIONOFINTEREST`
4. Use **Execute SQL** tab to run queries

#### Example Queries:

**Visit start times:**
```sql
SELECT
  DATETIME(ZENTRYDATE + 978307200, 'unixepoch') AS StartTime,
  ZLOCATIONLATITUDE,
  ZLOCATIONLONGITUDE
FROM ZRTLEARNEDVISITMO;
```

**Visit duration (minutes):**
```sql
SELECT
  (ZEXITDATE - ZENTRYDATE) / 60.0 AS DurationMinutes,
  DATETIME(ZENTRYDATE + 978307200, 'unixepoch') AS StartTime
FROM ZRTLEARNEDVISITMO
WHERE ZEXITDATE > 0;
```

**Known "locations of interest":**
```sql
SELECT
  ZIDENTIFIER,
  ZLOCATIONLATITUDE,
  ZLOCATIONLONGITUDE
FROM ZRTLEARNEDLOCATIONOFINTEREST;
```

> Most time values use **Mac Absolute Time** (since Jan 1, 2001). Convert by adding `978307200` seconds.

## 📊 Step 6: Supplementary Location Sources

- **Apple Health App**: GPS traces in workouts, steps, or routes
- **Apple Privacy Portal**: [https://privacy.apple.com](https://privacy.apple.com)
- **Google Timeline**: [https://timeline.google.com](https://timeline.google.com)
- **Significant Locations**: Settings → Privacy → Location Services → System Services

> Only some of this data is downloadable. Local backup remains the most complete source.

## 🚧 A Note on Professional Forensic Tools

High-end tools such as **Cellebrite UFED**, **Magnet AXIOM**, **GRAYKEY**, and **VERAKEY** offer:
- Full access to locked devices and deleted records
- Real-time device interrogation

But they are:
- **Extremely costly**
- **Restricted to law enforcement and certified labs**

For independent or civilian investigators, encrypted backups and open-source tools like `sqlitebrowser` remain viable and powerful alternatives.

## ✅ Conclusion

By combining a secure, encrypted backup with open source analysis tools, investigators can reliably access and interpret iOS geolocation data for legal, compliance, or investigative use. This method prioritizes data integrity, reproducibility, and accessibility without relying on proprietary or restricted platforms. When performed carefully, it supports evidentiary standards and forms the basis of defensible forensic review.

