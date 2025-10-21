# BPA-IPF GUI (Motif/X11) — WSL2 Build & Launch Notes

> ✅ **Status:** GUI successfully compiles and launches under WSL2 Ubuntu 22.04.
> ⚠️ **Limitation:** Segfaults after IPC init — GUI not fully functional (Motif-related).
> These notes document *everything that worked*, so it can be retried later if desired.

---

## 🧱 Environment

**System:**

* WSL2 Ubuntu 22.04
* Windows host running X11 server (e.g. X410, VcXsrv, or WSLg)

**Packages Installed**

```bash
sudo apt update
sudo apt install -y build-essential cmake gfortran git \
    libx11-dev libxt-dev libxm4 libmotif-dev libxmu-dev \
    x11-apps xterm uil strace
```

**Verified Motif Build Tools**

```bash
uil --version
xclock   # to verify X server connectivity
```

---

## 🧩 Build Procedure

```bash
git clone https://github.com/yourfork/bpa-ipf-tsp.git
cd bpa-ipf-tsp
mkdir build && cd build
cmake -DCMAKE_BUILD_TYPE=Release ..
make -j$(nproc)
ctest -C Release
```

✅ **All 8 CTest cases passed**

---

## 🪟 GUI Compilation (Motif UIDs)

Generate `gui.uid`:

```bash
cd ~/dev_for/bpa-ipf-tsp/gui
uil -I ../libgui -o gui.uid gui.uil
```

Result:

```
-rw-r--r-- 1 qgfl qgfl 1.2M gui.uid
```

---

## 🧭 GUI Environment Setup

The GUI relies on the **UIDPATH** and **IPF IPC** environment variables.

```bash
export IPFROOTDIR="$HOME/dev_for/bpa-ipf-tsp"
export IPFDIRS="$IPFROOTDIR"
export UIDPATH="$IPFROOTDIR/gui/%U"
export RUN_IPFSRV=YES
export IPF_SOCK=1024
export LANG=C
```

---

## ▶️ Launch Procedure

```bash
cd ~/dev_for/bpa-ipf-tsp/build/gui
[ -f gui.uid ] || ln -s ../../gui/gui.uid gui.uid
./gui
```

Expected output:

```
Warning: Urm__CW_FixupCallback: Callback routine 'quick_exit' not registered - MrmNOT_FOUND
  <bind> call status = 0 
IPC -- using socket number  1024
system command = < YES -socketid 1024 & >
status = 0
...
```

✅ **Motif window appears** (partial GUI visible)
⚠️ **Fails on IPC link / buffer overflow**

---

## 🧩 IPC Notes (`ipfsrv`)

You can manually launch the IPC server for testing:

```bash
ipfsrv -sock 1024 &
```

or via a wrapper script (`$HOME/bin/ipfsrv_shim`):

```bash
#!/usr/bin/env bash
exec "$HOME/dev_for/bpa-ipf-tsp/build/ipfsrv" -sock "$2"
```

Common logs:

```
SERVER: Initialize connection to socket id 1024
ipc failed on connect, status = -1, socket number = 1024
Warning - no IPC connection
Exiting...
```

---

## ⚠️ Known Limitations

| Area     | Symptom                   | Likely Cause                            |
| -------- | ------------------------- | --------------------------------------- |
| UID file | `MrmNOT_FOUND` initially  | Fixed by generating `gui.uid` via `uil` |
| IPC      | `Connection refused`      | `ipfsrv` not fully binding in WSL2      |
| GUI      | Segfault after connect    | Motif/Xt/Xm widget table mismatch       |
| Locale   | Warnings about I18N fonts | Harmless — `LANG=C` helps               |

---

## 🧹 Reset to Clean State

To revert environment and keep repository clean:

```bash
pkill ipfsrv 2>/dev/null
unset IPFROOTDIR IPFDIRS UIDPATH RUN_IPFSRV IPF_SOCK LANG
git status  # should show: nothing to commit, working tree clean
```

---

## 📎 References

* [BPA-IPF Official Docs – X Window GUI](https://bpa-ipf.readthedocs.io/en/latest/basic/x_window_gui.html)
* Source files involved:

  * `gui/gui.uil` (main UIL descriptor)
  * `libgui/*.u` (widget definitions)
  * `libgui/gui.c` (Motif GUI C entry point)
  * `ipfsrv` (IPC service used by GUI/CLI)

Perfect — here’s the final section you can append to the Markdown file (`docs/gui_wsl2_build_notes.md`) right after the References block:

---

## 🔁 Quick Retry (Future Builds)

If you ever want to retry launching the legacy Motif GUI after environment updates or dependency fixes, just run:

```bash
# 1. Prepare environment
export IPFROOTDIR="$HOME/dev_for/bpa-ipf-tsp"
export IPFDIRS="$IPFROOTDIR"
export UIDPATH="$IPFROOTDIR/gui/%U"
export RUN_IPFSRV=YES
export IPF_SOCK=1024
export LANG=C

# 2. Ensure gui.uid exists
cd "$IPFROOTDIR/gui"
[ -f gui.uid ] || uil -I ../libgui -o gui.uid gui.uil

# 3. Launch from build directory
cd "$IPFROOTDIR/build/gui"
[ -f gui.uid ] || ln -s ../../gui/gui.uid gui.uid
./gui
```

If the GUI opens but fails to connect to `ipfsrv`, try:

```bash
pkill ipfsrv 2>/dev/null
"$IPFROOTDIR/build/ipfsrv" -sock 1024 &
```

---

💡 *Tip:* You can wrap all of this into a helper script (`bin/run_gui.sh`) for future use:

```bash
#!/usr/bin/env bash
source "$HOME/.bashrc"
export IPFROOTDIR="$HOME/dev_for/bpa-ipf-tsp"
export IPFDIRS="$IPFROOTDIR"
export UIDPATH="$IPFROOTDIR/gui/%U"
export RUN_IPFSRV=YES
export IPF_SOCK=1024
export LANG=C
pkill ipfsrv 2>/dev/null
"$IPFROOTDIR/build/ipfsrv" -sock 1024 &
cd "$IPFROOTDIR/build/gui"
[ -f gui.uid ] || ln -s ../../gui/gui.uid gui.uid
./gui
```

Then just run:

```bash
~/bin/run_gui.sh
```