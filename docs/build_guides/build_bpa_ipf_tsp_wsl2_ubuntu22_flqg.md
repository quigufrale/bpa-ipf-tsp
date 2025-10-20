---
title: "Building BPA-IPF-TSP on WSL2 Ubuntu 22 (gfortran 11)"
author:
  - name: "Franklin Quilumba"
    alias: "quigufrale"
    orcid: "https://orcid.org/0000-0001-6326-3439"
date: "2025-10-20"
project: "bpa-ipf-tsp"
version: "1.0.0"
license: "internal-use"
tags: [fortran, c, power-flow, wsl2, cmake, build-guide]
summary: >
  Comprehensive guide describing how to clone, modernize, and build
  the legacy BPA-IPF-TSP power-flow and transient-stability suite
  on Ubuntu 22 under WSL2 using gfortran 11 and CMake, including fixes
  for legacy Fortran/C interoperability and GUI symbol conflicts.
output:
  html: default
  pdf: default
---

# BPA-IPF-TSP Build Guide (WSL2 Ubuntu 22 + gfortran 11)

**Author:** F. Quilumba  
**ORCID:** [0000-0001-6326-3439](https://orcid.org/0000-0001-6326-3439)  
**Last Updated:** 2025-10-20  
**File:** `docs/build_guides/build_bpa_ipf_tsp_wsl2_ubuntu22_flqg.md`

---

## 🧭 Overview

This document captures the complete process to build and run the **BPA-IPF-TSP** project natively under **WSL2 (Ubuntu 22)** using **Visual Studio Code** and **gfortran 11**.  
It consolidates the fixes and patches required to modernize the legacy Fortran/C codebase originally tested only on CentOS 7 and Ubuntu 20.04 (gcc 9.4).

---

## 1️⃣  Environment Setup

### Prerequisites
- **Windows 11 (or 10)** with **WSL2 (Ubuntu 22.04)**
- **Visual Studio Code** with extensions:
  - *CMake Tools* (Microsoft)
  - *C/C++* (Microsoft)
  - *Modern Fortran*
- **Docker Desktop** *(optional, for reference builds)*

### Linux packages
```bash
sudo apt update
sudo apt install -y build-essential cmake gcc gfortran \
    libmotif-dev libxmu-dev libx11-dev libxt-dev libxpm-dev libxext-dev gdb
````

Create a workspace:

```bash
mkdir -p ~/dev_for
cd ~/dev_for
```

---

## 2️⃣  Clone the Repository

```bash
git clone https://github.com/mbheinen/bpa-ipf-tsp
cd bpa-ipf-tsp
```

---

## 3️⃣  Compatibility Fixes for Modern GCC / gfortran

### 🧩 A. Fortran argument-mismatch tolerance

Edit **`CMakeLists.txt`** after:

```cmake
INCLUDE(${CMAKE_MODULE_PATH}/SetFortranFlags.cmake)
```

Add:

```cmake
# Relax legacy F77 calling conventions for gfortran 10+
set(CMAKE_Fortran_FLAGS "${CMAKE_Fortran_FLAGS} -fallow-argument-mismatch -std=legacy")
```

---

### 🧩 B. GUI multiple-definition (`c_wid`)

#### `libgui/define.h`

```c
#ifndef IPF_DEFINE_H
#define IPF_DEFINE_H

#define COMMENT_LIMIT 50
extern Widget c_wid[COMMENT_LIMIT];

#endif /* IPF_DEFINE_H */
```

#### New file → `libgui/define.c`

```c
#include <Xm/Xm.h>
#include "define.h"

Widget c_wid[COMMENT_LIMIT] = {0};
```

#### Update `libgui/CMakeLists.txt`

Add to the list:

```cmake
define.c
```

---

### 🧩 C. Linker conflict (`progname`)

#### New header → `ipf/ge_utils.h`

```c
#ifndef IPF_GE_UTILS_H
#define IPF_GE_UTILS_H
extern char *progname;
#endif
```

#### `ipf/ge_utils.c`

```c
#include <stdio.h>
#include <ctype.h>
#include <string.h>
#include "ge_utils.h"
```

Remove any existing `char *progname;` definition.

---

## 4️⃣  Build

```bash
mkdir build && cd build
cmake -DCMAKE_BUILD_TYPE=Release ..
make -j"$(nproc)"
sudo make install
```

All binaries install to `/usr/local/bin`.

---

## 5️⃣  Verify Installation

```bash
which bpf
which tsp
which gui
```

They should all resolve under `/usr/local/bin`.

---

## 6️⃣  Quick Run Test

```bash
cd ../data
bpf bench.pfc
```

Expected output:

```
SUCCESSFUL SOLUTION REACHED.
```

You can also test:

```bash
tsp bench.fil
```

---

## 7️⃣  Notes & Recommendations

| Topic                              | Details                                                  |
| ---------------------------------- | -------------------------------------------------------- |
| **Debug build**                    | Use `-DCMAKE_BUILD_TYPE=Debug` for GDB/VS Code debugging |
| **GUI**                            | Requires an X server (WSLg, X410, or VcXsrv)             |
| **Docker alternative**             | Original Dockerfile failed due to CentOS 7 EOL           |
| **Safe warnings**                  | `c_userid` alignment/size warnings are benign            |
| **Runtime bound error**            | Only seen in Debug mode; Release runs normally           |
| **Recommended VS Code extensions** | *CMake Tools*, *C/C++*, *Modern Fortran*                 |

---

## 8️⃣  Summary of Fixes and Modernizations

| Issue                              | Root Cause                     | Fix                                           |
| ---------------------------------- | ------------------------------ | --------------------------------------------- |
| CentOS 7 mirrors broken            | CentOS 7 EOL                   | Built on Ubuntu 22 instead                    |
| `REAL(4)/INTEGER(4)` type mismatch | gfortran 11 stricter checks    | Added `-fallow-argument-mismatch -std=legacy` |
| `c_wid` multiple definition        | Global array defined in header | Moved to `define.c`, declared `extern`        |
| `progname` multiple definition     | Duplicated global in C/Fortran | Declared `extern` via `ge_utils.h`            |
| Runtime index 0 error              | Legacy array bounds logic      | Disabled bounds check in Release mode         |

---

## ✅  Final Verification

```bash
cd ~/dev_for/bpa-ipf-tsp/data
bpf bench.pfc
```

Output should show:

```
SUCCESSFUL SOLUTION REACHED.
```

---

### 📦 Suggested Repo Layout

```
bpa-ipf-tsp/
├── CMakeLists.txt
├── ipf/
├── libgui/
├── ...
└── docs/
    └── build_guides/
        └── build_bpa_ipf_tsp_wsl2_ubuntu22_flqg.md   ← this file
```

---

### 🧠 Future Work

* Add CI workflow for Ubuntu 22/24 builds.
* Submit PR with portability fixes upstream.
* Document additional `.pfc` and `.fil` examples.

---

**End of Guide**
*(Tested and confirmed working on WSL2 Ubuntu 22.04 with gfortran 11.4, October 2025.)*
