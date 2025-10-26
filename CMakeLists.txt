# ---- Top-level CMake for BPA-IPF-TSP (out-of-source friendly) ----------------
# Requires generator expressions in add_test; 3.10+ is safe on modern systems.
cmake_minimum_required(VERSION 3.10)

project(bpa-ipf-tsp LANGUAGES Fortran C)
set(VERSION 0.0.2)

# ------------------------------------------------------------------------------
# Module path & common flags
# ------------------------------------------------------------------------------
set(CMAKE_MODULE_PATH "${CMAKE_SOURCE_DIR}/cmake/Modules/")
include(${CMAKE_MODULE_PATH}/SetFortranFlags.cmake)

# Relax legacy F77 calling conventions (gfortran 10+ stricter checks)
set(CMAKE_Fortran_FLAGS "${CMAKE_Fortran_FLAGS} -fallow-argument-mismatch -std=legacy")

# Handle pgf90 quirk
get_filename_component(FCNAME ${CMAKE_Fortran_COMPILER} NAME)
if(FCNAME STREQUAL "pgf90")
  unset(CMAKE_SHARED_LIBRARY_LINK_Fortran_FLAGS)
endif()

# GNU ar/ranlib wrappers for LTO (kept from original)
if(CMAKE_C_COMPILER_ID STREQUAL "GNU"
   AND NOT (CMAKE_C_COMPILER_VERSION VERSION_LESS 4.9.0)
   AND NOT OPENBSD AND NOT DRAGONFLY)
  if(DEFINED CHOST)
    set(CMAKE_AR "${CHOST}-gcc-ar")
    set(CMAKE_RANLIB "${CHOST}-gcc-ranlib")
  else()
    set(CMAKE_AR "gcc-ar")
    set(CMAKE_RANLIB "gcc-ranlib")
  endif()
endif()

# ------------------------------------------------------------------------------
# Output directories (build tree only)
# ------------------------------------------------------------------------------
# All executables -> <build>/bin
set(CMAKE_RUNTIME_OUTPUT_DIRECTORY ${CMAKE_BINARY_DIR}/bin)
# All shared/static libs -> <build>/lib
set(CMAKE_LIBRARY_OUTPUT_DIRECTORY ${CMAKE_BINARY_DIR}/lib)
set(CMAKE_ARCHIVE_OUTPUT_DIRECTORY ${CMAKE_BINARY_DIR}/lib)
# Fortran .mod files -> <build>/mod
set(CMAKE_Fortran_MODULE_DIRECTORY ${CMAKE_BINARY_DIR}/mod)

# ------------------------------------------------------------------------------
# Targets and source tree layout (no in-source build artifacts)
# ------------------------------------------------------------------------------
# Named libs/execs (kept for consistency with subdirs)
set(IPFLIB        ipf)
set(IPCLIB        ipc)
set(TSPLIB        libtsp)
set(CFLOWLIB      cflow)
set(IPFGUILIB     ipfgui)

set(BPFEXE        bpf)
set(IPFBATEXE     ipfbat)
set(IPFREPORTSEXE ipf_reports)
set(IPFTESTEXE    ipf_test)
set(IPFPLOTEXE    ipfplot)
set(IPFSRVEXE     ipfsrv)
set(GUIEXE        gui)
set(FINDOUTEXE    findout)
set(MIMICEXE      mimic)
set(PVCURVEEXE    pvcurve)
set(POSTPVCURVEEXE post_pvcurve)
set(QVCURVEEXE    qvcurve)
set(LINEFLOWEXE   lineflow)
set(GPFEXE        gpf)
set(PUFEXE        puf)
set(NETDATEXE     netdat)
set(IPFCUTEXE     ipfcut)
set(TSPEXE        tsp)

# Source folders
set(SRC         ${CMAKE_SOURCE_DIR})
set(SRCIPF      ${SRC}/ipf)
set(SRCIPC      ${SRC}/ipc)
set(SRCGUI      ${SRC}/gui)
set(SRCLIBTSP   ${SRC}/libtsp)
set(SRCLIBCFLOW ${SRC}/cflow)
set(SRCLIBGUI   ${SRC}/libgui)
set(SRCBPF      ${SRC}/bpf)
set(SRCIPFBAT   ${SRC}/ipfbat)
set(SRCIPFREPORTS ${SRC}/ipf_reports)
set(SRCIPFSRV   ${SRC}/ipfsrv)
set(SRCIPFTEST  ${SRC}/ipf_test)
set(SRCIPFPLOT  ${SRC}/ipfplot)
set(SRCGPF      ${SRC}/gpf)
set(SRCPUF      ${SRC}/puf)
set(SRCNETDAT   ${SRC}/netdat)
set(SRCIPFCUT   ${SRC}/ipfcut)
set(SRCTSP      ${SRC}/tsp)

# Libraries (now use default build-tree dirs; remove 2nd arg)
add_subdirectory(${SRCIPF})
add_subdirectory(${SRCIPC})
add_subdirectory(${SRCLIBTSP})
add_subdirectory(${SRCLIBGUI})
add_subdirectory(${SRCLIBCFLOW})

# Executables (no in-source bin; executables will land in <build>/bin)
add_subdirectory(${SRCBPF})
add_subdirectory(${SRCIPFBAT})
add_subdirectory(${SRCIPFREPORTS})
add_subdirectory(${SRCIPFSRV})
add_subdirectory(${SRCGUI})
add_subdirectory(${SRCIPFTEST})
add_subdirectory(${SRCIPFPLOT})
# add_subdirectory(${SRCGPF})
add_subdirectory(${SRCPUF})
add_subdirectory(${SRCNETDAT})
# add_subdirectory(${SRCIPFCUT})
add_subdirectory(${SRCTSP})

# ------------------------------------------------------------------------------
# Tests (use generator expressions to call targets; workdir = source data dir)
# ------------------------------------------------------------------------------
enable_testing()

# Helper var for data folder
set(DATA_DIR ${CMAKE_SOURCE_DIR}/data)

add_test(NAME bpf-bench.pfc
         COMMAND $<TARGET_FILE:bpf> bench.pfc
         WORKING_DIRECTORY ${DATA_DIR})

add_test(NAME bpf-new_system1.pfc
         COMMAND $<TARGET_FILE:bpf> new_system1.pfc
         WORKING_DIRECTORY ${DATA_DIR})

add_test(NAME bpf-new_system2.pfc
         COMMAND $<TARGET_FILE:bpf> new_system2.pfc
         WORKING_DIRECTORY ${DATA_DIR})

add_test(NAME bpf-test_gov.pfc
         COMMAND $<TARGET_FILE:bpf> test_gov.pfc
         WORKING_DIRECTORY ${DATA_DIR})

add_test(NAME bpf-testdc3.pfc
         COMMAND $<TARGET_FILE:bpf> testdc3.pfc
         WORKING_DIRECTORY ${DATA_DIR})

add_test(NAME ipf_reports-bench-selected-bus-branches
         COMMAND /bin/bash -c "echo -e \"bench.bse\nO\nT\n1\nBUS1,230,1\nQ\nQ\nQ\" | $<TARGET_FILE:ipf_reports>"
         WORKING_DIRECTORY ${DATA_DIR})

add_test(NAME ipf_reports-bench-transmission-line-loadings
         COMMAND /bin/bash -c "echo -e \"bench.bse\nO\nT\n5\nAll\n95\n\nQ\nQ\nQ\" | $<TARGET_FILE:ipf_reports>"
         WORKING_DIRECTORY ${DATA_DIR})

add_test(NAME ipf_reports-bench-transformer-loadings
         COMMAND /bin/bash -c "echo -e \"bench.bse\nO\nT\n6\nAll\n95\n0\nQ\nQ\nQ\" | $<TARGET_FILE:ipf_reports>"
         WORKING_DIRECTORY ${DATA_DIR})
