# Ransomware Behaviour Simulator

An educational, sandboxed tool that simulates ransomware techniques for academic study and detection research. Built in Python for Kali Linux.

> **This tool is for educational purposes only.**
> No real encryption is used. No network communication occurs. All activity is strictly contained within a sandboxed test directory. It cannot affect files outside the sandbox.

---

## What It Does

The simulator replicates the core stages of the ransomware kill chain in a fully controlled environment:

| Stage | What Happens |
| --- | --- |
| **Enumeration** | Walks the sandbox directory and collects target files by extension |
| **Fake Encryption** | XOR-encodes file contents and renames them with a `.locked` extension |
| **Key Drop** | Writes a fake `master.key` file simulating key artefact behaviour |
| **Ransom Note** | Drops `README_RESTORE.txt` into every affected directory |
| **Detection** | A separate monitor watches for suspicious filesystem events via inotify |
| **Dashboard** | A live PyQt5 GUI shows file events, threat level, and alerts in real time |

---

## Project Structure

```
ransomware_simulator/
├── main.py                    ← Simulator entry point (CLI)
├── monitor.py                 ← Standalone console detection monitor
├── config.py                  ← All settings (XOR key, extensions, thresholds)
├── requirements.txt
│
├── modules/
│   ├── enumerator.py          ← Phase 2 — file discovery and metadata
│   ├── encryptor.py           ← Phase 3 — XOR encoding and restore
│   ├── note_dropper.py        ← Phase 4 — ransom note generation
│   └── detector.py            ← Phase 5 — watchdog detection engine
│
├── dashboard/
│   └── dashboard.py           ← Phase 5B — PyQt5 live detection GUI
│
├── utils/
│   ├── logger.py              ← Shared logger (console + file)
│   └── sandbox_builder.py     ← Builds and manages the sandbox
│
├── sandbox/                   ← Isolated test directory (auto-generated)
└── logs/
    └── simulator.log          ← Full run log
```

---

## Requirements

- Python 3.10+
- Kali Linux (or any Linux with inotify support)

Install dependencies:

```bash
pip install -r requirements.txt
```

```
watchdog>=3.0.0
PyQt5>=5.15.0
```

---

## Quick Start

```bash
# 1. Build the sandbox with dummy files
python main.py --reset

# 2. Start the detection monitor (separate terminal)
python monitor.py

# 3. Run the full simulation (separate terminal)
python main.py --run

# 4. Restore everything back to original state
python main.py --restore
```

---

## Simulator — main.py

```
python main.py [-h] (--run | --dry-run | --restore | --reset | --status)
```

### Commands

```
--run         Run the full simulation
              Enumerate → XOR encrypt → rename to .locked → drop ransom notes

--dry-run     Enumerate target files only
              Lists all files that would be affected — no changes made

--restore     Reverse the simulation
              XOR decodes all .locked files, restores original filenames,
              removes ransom notes and master.key

--reset       Wipe and rebuild the sandbox
              Deletes the existing sandbox/ and repopulates it with fresh
              dummy files across 5 typed folders (documents, images, data,
              code, personal). Run this before starting a new demo session.

--status      Print current sandbox state
              Shows total files, locked files, ransom notes, and clean files

-h, --help    Show help message
```

### Examples

```bash
# See what would be targeted without touching anything
python main.py --dry-run

# Run the full kill chain
python main.py --run

# Check state mid-simulation
python main.py --status

# Undo everything
python main.py --restore

# Start fresh
python main.py --reset
```

---

## Detection Monitor — monitor.py

Runs in a **separate terminal** and watches the sandbox for suspicious activity using watchdog (inotify on Linux). Events are colour-coded by threat level.

```
python monitor.py [--summary]
```

```
--summary     Print a full event report on exit (Ctrl+C)
```

### Threat Levels

| Symbol | Level | Trigger |
| --- | --- | --- |
| `ℹ` | INFO | Any file modification or rename |
| `⚠` | SUSPECT | File renamed to `.locked` extension |
| `🚨` | ALERT | Ransom note / key file created, or mass rename/modify threshold crossed |

### Alert Threshold

An ALERT fires when **5 or more** file events occur within a **2 second** window. Both values are configurable in `config.py`:

```python
DETECTION_THRESHOLD_COUNT   = 5   # events
DETECTION_THRESHOLD_SECONDS = 2   # window in seconds
```

---

## Detection Dashboard — dashboard/dashboard.py

A PyQt5 GUI that displays live detection data across three panels.

```bash
python dashboard/dashboard.py
```

| Panel | Content |
| --- | --- |
| **Files Modified** | Scrolling list of every file event with timestamp |
| **Threat Level** | Live suspicious event counter — escalates green → amber → red |
| **Alerts Triggered** | Highlighted log of all ALERT-level events |

---

## Correct Usage Order

> The monitor and dashboard must be started **after** `--reset` and **before** `--run`.
> If `--reset` is run while the monitor is active, watchdog loses its inotify watch on the recreated directory.

```bash
# Correct order every time
python main.py --reset            # build sandbox first
python monitor.py                 # start monitor second
python main.py --run              # run simulator third
```

To run multiple demos without restarting the monitor, use `--restore` instead of `--reset`:

```bash
python main.py --restore          # clean up — monitor stays live
python main.py --run              # run again immediately
```

---

## Configuration — config.py

All behaviour is controlled from a single config file.

| Setting | Default | Description |
| --- | --- | --- |
| `SANDBOX_DIR` | `./sandbox` | Absolute path to the sandboxed test directory |
| `XOR_KEY` | `0x5A` | Single-byte XOR key used for fake encryption |
| `LOCKED_EXTENSION` | `.locked` | Extension appended to encoded files |
| `KEY_FILE_NAME` | `master.key` | Name of the fake key artefact |
| `TARGET_EXTENSIONS` | `.txt .docx .pdf .jpg ...` | File types targeted by the simulator |
| `RANSOM_NOTE_FILENAME` | `README_RESTORE.txt` | Name of the dropped ransom note |
| `DETECTION_THRESHOLD_COUNT` | `5` | Events required to trigger a mass-activity alert |
| `DETECTION_THRESHOLD_SECONDS` | `2` | Time window for threshold detection |

---

## Sandbox Contents

The sandbox is populated with 17 dummy files across 5 folders:

```
sandbox/
├── documents/       report_Q1.txt, meeting_notes.txt, contract_draft.docx ...
├── images/          photo_vacation.jpg, screenshot.png, profile_pic.jpeg
├── data/            users.csv, config.json, inventory.xlsx
├── code/            backup_script.py, index.html, data_export.xml
└── personal/        diary_entry.txt, passwords_hint.txt, todo_list.txt
```

No real personal data. All content is fabricated for simulation purposes.

---

## Real-World Comparisons

| Feature | WannaCry | LockBit | This Simulator |
| --- | --- | --- | --- |
| Encryption | AES-128 + RSA-2048 | AES-256 + RSA-4096 | XOR 0x5A (educational) |
| Extension | `.WNCRY` | `.lockbit` | `.locked` |
| Ransom note | `@Please_Read_Me@.txt` | `LockBit_Ransomware.hta` | `README_RESTORE.txt` |
| Note placement | Every folder | Every folder | Every folder |
| Network C2 | Yes | Yes | None |
| Real payload | Yes | Yes | No  |

---

## Logs

All events are written to `logs/simulator.log` with full timestamps. The log persists across runs and captures output from all modules.

```bash
tail -f logs/simulator.log    # live log stream
cat logs/simulator.log        # full history
```

---

## Academic Context

This project demonstrates:

- **Ransomware kill chain** — enumeration, encryption, exfiltration simulation, and note dropping as discrete, observable stages
- **Filesystem internals** — directory traversal, file metadata, inode-level monitoring via inotify
- **Cryptography concepts** — XOR as a reversible encoding scheme versus AES/RSA as used in real ransomware
- **Detection heuristics** — sliding time-window event burst detection mirroring real SIEM/EDR logic
- **Sandboxing** — hard path boundary enforcement using `os.path.realpath` to prevent escape via symlinks or relative paths
