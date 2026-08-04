# Entry-P01NT Workshop Lab Binaries

This directory contains the lab binary files for the Entry-P01NT workshop.

## Structure

```
static/assets/binaries/
├── 0x1-intro-re.7z          # Lesson 0x1 lab (if applicable)
├── 0x2-fundamentals.7z      # Lesson 0x2 lab (if applicable)
├── 0x3-windows-pe.7z        # Lesson 0x3 lab (if applicable)
├── 0x4-c-constructs.7z      # Lesson 0x4 lab (if applicable)
├── 0x5-cyber-lifecycle.7z   # Lesson 0x5 lab (if applicable)
├── 0x6-reconnaissance.7z    # Lesson 0x6 lab (if applicable)
├── 0x7-re-toolkit.7z        # Lesson 0x7 lab (if applicable)
├── 0x8-thinking-lab.7z      # Lesson 0x8 lab (Static Lab 🔍)
├── 0x9-static-lab.7z        # Lesson 0x9 lab (Static Lab 🔍)
├── 0xA-dynamic-lab.7z       # Lesson 0xA lab (Dynamic Lab ⚙️)
└── 0xB-after-that.7z        # Lesson 0xB lab (if applicable)
```

## Password

All archives are encrypted with the password: **`p01nt`**

## ⚠️ Safety Warning

**RUN THESE BINARIES ONLY IN AN ISOLATED VIRTUAL MACHINE**

- These are real malware samples / challenge binaries
- They may attempt network connections, file modifications, or other malicious actions
- Use a VM with no network access (host-only or disconnected)
- Take snapshots before running
- Do NOT run on your host machine

## Lab Lessons with Binaries

Based on the workshop curriculum, the following lessons include lab binaries:

| Lesson | Hex | Stage | Description |
|--------|-----|-------|-------------|
| 0x8 | 0x8 | Practice | The Reverse Engineer's Mindset — Static Lab 🔍 |
| 0x9 | 0x9 | Practice | Static Lab 🔍 — Analyzing code without execution |
| 0xA | 0xA | Practice | Dynamic Lab ⚙️ — Monitoring program during execution |

## Adding New Lab Binaries

1. Place the `.7z` file in this directory
2. Update the lesson front matter:
   ```yaml
   lab: "/assets/binaries/your-lab-file.7z"
   labPassword: "p01nt"
   ```
3. The workshop template will automatically show a download card

## Extraction

```bash
# Using 7z (cross-platform)
7z x lab-file.7z -pp01nt

# Using 7-Zip GUI
# Right-click → Extract → Enter password: p01nt
```

---

**Note**: The actual binary files are not committed to this repository due to security policies. They should be added manually before deployment or distributed through a separate secure channel.