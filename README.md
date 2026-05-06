# IODEVOPS
Restart Autosys Service


Great practical question. Here are all the ways to verify on the server:

---

## 1. Check if systemd is the Init System at all

```bash
ps -p 1 -o comm=
```
If output is `systemd` → systemd is running. If it's `init` or `upstart` → systemd module **will not work at all**.

```bash
systemctl --version
```
If this errors → no systemd. Game over for `ansible.builtin.systemd`.

---

## 2. Check if Autosys is Registered as a systemd Unit

```bash
# Quick existence check
systemctl list-units --type=service | grep -i autosys

# More thorough — includes inactive/disabled units
systemctl list-unit-files --type=service | grep -i autosys

# Direct status hit
systemctl status autosys
```

**What the outputs mean:**

| Output | Meaning |
|---|---|
| `autosys.service   active running` | ✅ systemd manages it — module works |
| `autosys.service   inactive dead` | ✅ Unit exists but stopped — module works |
| `Unit autosys.service could not be found` | ❌ Not registered — use `unisrvcntr` |

---

## 3. Check if the Unit File Actually Exists on Disk

```bash
# Standard locations
ls -la /etc/systemd/system/ | grep -i autosys
ls -la /usr/lib/systemd/system/ | grep -i autosys
ls -la /lib/systemd/system/ | grep -i autosys

# Let systemd tell you where it found the file
systemctl show autosys --property=FragmentPath
```

If `FragmentPath=` is **empty** → no unit file exists.

---

## 4. Read the Unit File (if it exists)

```bash
systemctl cat autosys
```

This shows you exactly what `ExecStart` and `ExecStop` are calling. You'll see one of two things:

**Good — wraps unisrvcntr:**
```ini
ExecStart=/opt/CA/.../unisrvcntr start all
ExecStop=/opt/CA/.../unisrvcntr stop all
```

**Bad — wrong service name or placeholder:**
```ini
ExecStart=/bin/true
```
This means someone created a dummy unit — `systemd` won't actually control Autosys properly.

---

## 5. Verify unisrvcntr Directly

```bash
# Find the binary
which unisrvcntr
find / -name "unisrvcntr" 2>/dev/null

# Check if autosys user environment is set up
su - autosys -c 'env | grep -E "AUTOSYS|AUTOUSER|PATH"'

# Run status directly
su - autosys -c 'unisrvcntr status all'
```

Expected healthy output:
```
WA_SCHEDULER    RUNNING    pid=4821
WA_SERVER       RUNNING    pid=4822
WA_DATABASE     RUNNING    pid=4823
```

---

## 6. One-Shot Diagnostic Script (run as root)

Paste this directly on the server for a full picture:

```bash
#!/bin/bash
echo "====== INIT SYSTEM ======"
ps -p 1 -o comm=
systemctl --version 2>/dev/null | head -1

echo ""
echo "====== SYSTEMD UNIT CHECK ======"
systemctl list-unit-files --type=service 2>/dev/null | grep -i autosys \
  || echo "NO autosys systemd unit found"

echo ""
echo "====== UNIT FILE LOCATION ======"
systemctl show autosys --property=FragmentPath 2>/dev/null

echo ""
echo "====== UNISRVCNTR BINARY ======"
which unisrvcntr 2>/dev/null || find /opt /usr -name "unisrvcntr" 2>/dev/null

echo ""
echo "====== AUTOSYS USER ENV ======"
su - autosys -c 'env | grep -E "AUTOSYS|AUTOUSER"' 2>/dev/null

echo ""
echo "====== UNISRVCNTR STATUS ======"
su - autosys -c 'unisrvcntr status all' 2>/dev/null
```

---

## Decision Tree Based on What You Find

```
systemctl --version  →  errors?
        │
        ├── YES → No systemd at all
        │         Use: ansible.builtin.shell + unisrvcntr only
        │
        └── NO  → systemd present
                  │
                  systemctl status autosys  →  "could not be found"?
                  │
                  ├── YES → No unit file
                  │         Use: ansible.builtin.shell + unisrvcntr
                  │
                  └── NO  → Unit exists
                            │
                            systemctl cat autosys  →  ExecStop calls unisrvcntr?
                            │
                            ├── YES → ansible.builtin.systemd works ✅
                            │
                            └── NO  → Dummy unit — use unisrvcntr directly ⚠️
```

---
