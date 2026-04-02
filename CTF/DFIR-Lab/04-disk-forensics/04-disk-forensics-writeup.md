# Challenge 04: Disk Forensics

## 🧠 Scenario

Two disk images were recovered from a suspect workstation:

- `suspicious.dd` — A 32MB USB drive image
- `deleted-files.img` — A 16MB image containing deleted files

The objective was to analyze both images for evidence of data theft and attack planning.

---

## 🛠 Tools Used

- mount
- find
- ls
- cat
- strings
- foremost
- base64

---

# 🔎 Challenge 4.1 — Hidden Stolen Data File

### 🧩 Approach

To find the hidden stolen data file, the `suspicious.dd` image was mounted read-only and a recursive listing was performed. A hidden directory `.secret` was discovered inside `/mnt/documents/`, which is not visible in a normal listing. Inside it was `stolen-data.csv`, making the full path `/documents/.secret/stolen-data.csv`.

### 🛠 Commands

```bash
sudo mount -o ro,loop suspicious.dd /mnt
ls -laR /mnt
```

### 🎯 Result

FLAG{/documents/.secret/stolen-data.csv}

---

# 🔎 Challenge 4.2 — Cleanup Script Analysis

### 🧩 Approach

The cleanup script at `/mnt/tmp/cleanup.sh` was read using `cat`. It contained a `shred` command targeting `/var/log/*`, meaning the attacker attempted to destroy all log files using a glob pattern to cover their tracks.

### 🛠 Commands

```bash
cat /mnt/tmp/cleanup.sh
```

### 🎯 Result

FLAG{/var/log/*}

---

# 🔎 Challenge 4.3 — Flag in Recovered Image

### 🧩 Approach

Foremost was used to carve image files from `suspicious.dd`. A PNG file was successfully recovered at `/tmp/carved-sus/png/00012416.png`. Since the lab machine had no GUI or X server, the image could not be opened directly. Instead it was base64-encoded on the remote machine, copied to the local Kali machine, decoded using a heredoc to avoid invalid input errors, and then opened to reveal the flag.

### 🛠 Commands

```bash
# Carve image files from the disk image
foremost -t png,jpg,bmp,gif -i suspicious.dd -o /tmp/carved-sus

# Check what was carved
ls -laR /tmp/carved-sus

# Encode the carved image to base64 for transfer
base64 /tmp/carved-sus/png/00012416.png

# On local Kali machine, decode the base64 output
cat > /tmp/flag.b64 << 'EOF'
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
EOF
base64 -d /tmp/flag.b64 > ~/04-4_3-recovered-img.png

# Open the image
xdg-open ~/04-4_3-recovered-img.png
```

### 🎯 Result

FLAG{recovered_evidence_img}

---

# 🔎 Challenge 4.4 — Deleted Credentials Recovery

### 🧩 Approach

Even though `creds.txt` had been deleted from the disk image, the file contents were not overwritten, meaning the raw data was still present in the image. Using `strings` on `deleted-files.img` and filtering for credential-related keywords, the contents of the deleted file were recovered, showing `admin:FLAG{deleted_but_not_gone}`. This demonstrates that deleting a file does not immediately erase its data from disk, making forensic recovery possible.

### 🛠 Commands

```bash
strings deleted-files.img | grep -i "pass\|admin\|user\|cred"
```

### 🎯 Result

FLAG{deleted_but_not_gone}

---

# 🔎 Challenge 4.5 — Exfiltration Timeline

### 🧩 Approach

Using `strings` on `deleted-files.img` and filtering for keywords like `exfil`, the contents of a deleted file called `plan.txt` were recovered. The file was part of `OPERATION NIGHTFALL - CONFIDENTIAL`. Even though the file had been deleted, its raw data was still present in the disk image. The plan revealed that the exfiltration was scheduled for `2025-11-15 02:00 UTC` via a DNS tunnel, targeting customer database and financial records. This again demonstrates that deleting files does not erase their contents from disk immediately, making forensic recovery possible.

### 🛠 Commands

```bash
strings deleted-files.img | grep -i -A5 -B5 "exfil\|2025\|midnight\|transfer\|send"
```

### 🎯 Result

FLAG{2025-11-15 02:00 UTC}

---

## ⚠️ Challenges Faced

- Lack of GUI in the lab environment prevented direct viewing of images
- Foremost initially appeared to hang during carving
- Mounted disk images did not reveal deleted files through normal listing
- Required switching to raw disk analysis (`strings`) to recover deleted artifacts

---

## 🧠 Key Takeaways

- Hidden files can be discovered through recursive directory analysis
- Deleted files remain recoverable until overwritten
- File carving enables recovery of artifacts from raw disk images
- Base64 is useful for transferring binary files from restricted environments
- Attack timelines can be reconstructed from residual disk data

---

## 🚀 Conclusion

This challenge demonstrated a full disk forensics workflow including mounting disk images, recursive file discovery, file carving with foremost, and recovering deleted artifacts using strings. The investigation uncovered evidence of data theft, a cleanup script designed to destroy evidence, a hidden flag inside a carved image, deleted credentials, and an attacker's exfiltration plan.
