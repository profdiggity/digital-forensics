# 📱 iOS Geolocation Forensics: Backup, Discovery, &amp; Analysis

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

## 🔬 Step 5: Analyze Routined Location Data (Learned Visits)

The `routined` system on iOS stores location patterns in standard SQLite format and is the most accessible geolocation source from a decrypted backup.

### Summary of Tools

| Tool                      | Use                                          |
|---------------------------|-----------------------------------------------|
| `sqlitebrowser`           | Explore `.sqlite` files                      |

### 🔧 Tool: `DB Browser for SQLite`

Install on Debian:
```bash
sudo apt install sqlitebrowser
```

### Files to Analyze:
```
/private/var/mobile/Library/Caches/com.apple.routined/Cache.sqlite
/private/var/mobile/Library/Caches/com.apple.routined/Local.sqlite
```

### Tables of Interest:
- `ZRTLEARNEDVISITMO` – captures visit start/end time and location
- `ZRTLEARNEDLOCATIONOFINTEREST` – stores significant places

### Example Queries:

**Visit start times:**
```sql
SELECT
  DATETIME(ZENTRYDATE + 978307200, 'unixepoch') AS StartTime,
  ZLOCATIONLATITUDE,
  ZLOCATIONLONGITUDE
FROM ZRTLEARNEDVISITMO;
```

**Visit duration (in minutes):**
```sql
SELECT
  (ZEXITDATE - ZENTRYDATE) / 60.0 AS DurationMinutes,
  DATETIME(ZENTRYDATE + 978307200, 'unixepoch') AS StartTime
FROM ZRTLEARNEDVISITMO
WHERE ZEXITDATE > 0;
```

**Known locations of interest:**
```sql
SELECT
  ZIDENTIFIER,
  ZLOCATIONLATITUDE,
  ZLOCATIONLONGITUDE
FROM ZRTLEARNEDLOCATIONOFINTEREST;
```

### ⏱️ Timestamp Reference

Apple stores timestamps in **Mac Absolute Time** or "Apple Cocoa Core Data Timestamp" (number of seconds since midnight Jan 1, 2001 UTC). To convert to standard Unix timestamp (seconds since midnight Jan 1, 1970 UTC), **add `978307200` seconds**:

- [Mac Absolute Time Converter](https://www.epochconverter.com/coredata)
- [Unix Time](https://en.wikipedia.org/wiki/Unix_time)

## 🗺️ Step 6: Analyze Apple Maps History & Tile Cache

Apple Maps stores history and cached imagery in a mix of SQLite databases and binary formats. These require both traditional and custom analysis tools.

### Summary of Tools

| Tool                      | Use                                          |
|---------------------------|-----------------------------------------------|
| `sqlitebrowser`           | Explore `.sqlite` files                      |
| `plistutil` / `biplist`   | Convert `.plist` to readable format          |
| `python3 + sqlite3`       | Extract raw blobs from Apple Maps data       |
| `OpenCV`                  | Auto-stitch satellite map tiles              |
| `xxd`, `file`, `magic`    | Analyze raw binary data types (optional)     |

### Files of Interest:
```
/private/var/mobile/Containers/Data/Application/[APPGUID]/Library/Maps/GeoHistory.mapsdata
/private/var/mobile/Containers/Data/Application/[APPGUID]/Library/Maps/GeoBookmarks.plist
/private/var/mobile/Library/Caches/com.apple.geod/MapTiles.sqlite
```

### 🔧 A. Convert Bookmarks from PLIST

Install:
```bash
sudo apt install libplist-utils
```

Convert:
```bash
plistutil -i GeoBookmarks.plist -o GeoBookmarks.xml
less GeoBookmarks.xml
```

Or via Python:
```bash
pip install biplist
python3 -c "import biplist; print(biplist.readPlist('GeoBookmarks.plist'))"
```

### 🔍 B. Extract Map Images from SQLite (MapTiles.sqlite or GeoHistory.mapsdata)

Use this script to automatically:
- Extract image or vector data from any SQLite BLOBs
- Detect whether the blob is JPEG, VMP4, or unknown
- Save the blobs to files

#### 🐍 Python Script — Smart BLOB Extractor

```python
import sqlite3, os

def detect_blob_type(blob):
    if blob.startswith(b'\xFF\xD8'):
        return 'jpg'
    elif b'VMP4' in blob[:32]:
        return 'vmp4'
    else:
        return 'bin'

def extract_blobs_smart(db_path, output_dir="extracted_blobs"):
    os.makedirs(output_dir, exist_ok=True)
    conn = sqlite3.connect(db_path)
    cursor = conn.cursor()
    tables = cursor.execute("SELECT name FROM sqlite_master WHERE type='table';").fetchall()

    for table_name, in tables:
        try:
            cursor.execute(f"PRAGMA table_info({table_name})")
            columns = [col[1] for col in cursor.fetchall()]
            blob_columns = [c for c in columns if 'data' in c.lower() or 'blob' in c.lower()]

            for col in blob_columns:
                cursor.execute(f"SELECT rowid, {col} FROM {table_name}")
                for rowid, blob in cursor.fetchall():
                    if blob and isinstance(blob, bytes):
                        ext = detect_blob_type(blob)
                        path = os.path.join(output_dir, f"{table_name}_{rowid}.{ext}")
                        with open(path, 'wb') as f:
                            f.write(blob)
        except Exception as e:
            print(f"Skipped table '{table_name}': {e}")
            continue

    conn.close()
    print(f"✅ BLOB extraction complete: saved to '{output_dir}'.")

# Usage:
extract_blobs_smart("MapTiles.sqlite")
```

### 🧵 C. Automatically Stitch JPEG Tiles (OpenCV)

Install:
```bash
pip install opencv-contrib-python
```

Run this to auto-merge tile images into a single stitched map:

```python
import cv2, os, glob

def stitch_images(image_dir, output_path="stitched_map.jpg"):
    image_paths = sorted(glob.glob(os.path.join(image_dir, "*.jpg")))
    if len(image_paths) < 2:
        print("Need at least 2 JPEGs to stitch.")
        return

    images = [cv2.imread(path) for path in image_paths if cv2.imread(path) is not None]
    stitcher = cv2.Stitcher_create() if hasattr(cv2, 'Stitcher_create') else cv2.createStitcher()
    status, stitched = stitcher.stitch(images)

    if status == cv2.Stitcher_OK:
        cv2.imwrite(output_path, stitched)
        print(f"✅ Stitched map saved to: {output_path}")
    else:
        print(f"❌ Stitching failed with status code {status}")

# Usage:
stitch_images("extracted_blobs")
```

## 📊 Step 7: Supplementary Location Sources

- **Apple Health App**: GPS traces in workouts, steps, or routes
- **Apple Privacy Portal**: [https://privacy.apple.com](https://privacy.apple.com)
- **Google Timeline**: [https://timeline.google.com](https://timeline.google.com)
- **Significant Locations**: Settings → Privacy → Location Services → System Services

> Only some of this data is downloadable. Local backup remains the most complete source.

## 🚧 A Note on Police Forensic Tools

High-end tools such as **Cellebrite UFED**, **Magnet AXIOM**, **GRAYKEY**, and **VERAKEY** offer:
- Full access to locked devices and deleted records
- Real-time device interrogation

But they are:
- **Extremely costly**
- **Restricted to law enforcement and certified labs**

For independent or civilian investigators, encrypted backups and open-source tools like `sqlitebrowser` remain viable and powerful alternatives.

## ✅ Conclusion

By combining a secure, encrypted backup with open source analysis tools, investigators can reliably access and interpret iOS geolocation data for legal, compliance, or investigative use. This method prioritizes data integrity, reproducibility, and accessibility without relying on proprietary or restricted platforms. When performed carefully, it supports evidentiary standards and forms the basis of defensible forensic review.
