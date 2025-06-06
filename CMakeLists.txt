cmake_minimum_required (VERSION 3.16)
project (qcint C)
set(qcint_VERSION_MAJOR "6")
set(qcint_VERSION_MINOR "1")
set(qcint_VERSION_PATCH "2")
set(qcint_VERSION_TWEAK "0")
set(qcint_VERSION "${qcint_VERSION_MAJOR}.${qcint_VERSION_MINOR}.${qcint_VERSION_PATCH}")
set(cint_SOVERSION "${qcint_VERSION_MAJOR}")

#set(CMAKE_C_FLAGS "${CMAKE_C_FLAGS} -g -O2 -DNDEBUG")
if ("${CMAKE_BUILD_TYPE}" STREQUAL "")
  set(CMAKE_BUILD_TYPE RELWITHDEBINFO)
endif()
set(CMAKE_VERBOSE_MAKEFILE OFF)
option(BUILD_MARCH_NATIVE "gcc flag -march=native" on)
if(BUILD_MARCH_NATIVE)
  message("Build with -march=native")
  set(CMAKE_C_FLAGS "${CMAKE_C_FLAGS} -march=native -O2")
else()
  # This appears to be the lowest version that compiles
  message("Build with -march=sse3")
  set(CMAKE_C_FLAGS "${CMAKE_C_FLAGS} -msse3 -O2")
endif()
if ("${CMAKE_C_COMPILER_ID}" STREQUAL "Intel")
  set(CMAKE_C_FLAGS "${CMAKE_C_FLAGS} -unroll-aggressive -ipo ")
  set(CMAKE_C_FLAGS "${CMAKE_C_FLAGS} -fno-math-errno")
else()
  set(CMAKE_C_FLAGS "${CMAKE_C_FLAGS} -funroll-loops -ftree-vectorize -ffast-math")
  set(CMAKE_C_FLAGS "${CMAKE_C_FLAGS} -fno-math-errno")
endif()

cmake_policy(SET CMP0042 NEW)
include(GNUInstallDirs)
#set(CMAKE_INCLUDE_CURRENT_DIR ON)

list(APPEND CMAKE_MODULE_PATH "${PROJECT_SOURCE_DIR}/cmake")
find_package(QUADMATH)
if(QUADMATH_FOUND)
  set(HAVE_QUADMATH_H 1 CACHE INTERNAL "Have QUADMATH")
endif()

if(${CMAKE_MAJOR_VERSION}.${CMAKE_MINOR_VERSION} VERSION_GREATER 3.12)
  cmake_policy(SET CMP0075 NEW)
endif()

include(CheckFunctionExists)
set(CMAKE_REQUIRED_INCLUDES math.h)
set(CMAKE_REQUIRED_LIBRARIES m)
check_function_exists(expl HAVE_EXPL)
check_function_exists(sqrtl HAVE_SQRTL)
#set(CMAKE_REQUIRED_INCLUDES quadmath.h)
#set(CMAKE_REQUIRED_LIBRARIES quadmath)
#check_function_exists(fabsq HAVE_QUADMATH_H)

include(CheckNativeVectorization)
check_native_vectorization()
message("sse3 works: ${SSE3_WORKS}")
message("avx works: ${AVX_WORKS}")
message("avx512f works: ${AVX512F_WORKS}")
if(AVX512F_WORKS)
  set(QCINT_VECTOR_LEVEL "avx512f")
  set(QCINT_SIMDD 8)
elseif(AVX_WORKS)
  set(QCINT_VECTOR_LEVEL "avx")
  set(QCINT_SIMDD 4)
elseif(SSE3_WORKS)
  set(QCINT_VECTOR_LEVEL "sse3")
  set(QCINT_SIMDD 2)
else()
  message(FATAL_ERROR "SSE3 is the minimum supported instruction set")
endif()

message("Will compile with vectorization width: ${QCINT_VECTOR_LEVEL}, SIMDD=${QCINT_SIMDD}")

configure_file(
  "${PROJECT_SOURCE_DIR}/src/cint_config.h.in"
  "${PROJECT_BINARY_DIR}/src/cint_config.h")

configure_file(
  "${PROJECT_SOURCE_DIR}/include/cint.h.in"
  "${PROJECT_BINARY_DIR}/include/cint.h")

set(cintSrc
  src/breit.c src/c2f.c src/cart2sph.c src/cint1e.c src/cint2c2e.c
  src/cint2e.c src/cint3c1e.c src/cint3c2e.c src/cint_bas.c src/fblas.c
  src/g1e.c src/g2e.c src/g2e_simd1.c src/g3c1e.c src/g3c2e.c
  src/gout2e.c src/misc.c src/optimizer.c
  src/fmt.c src/rys_wheeler.c src/eigh.c src/rys_roots.c src/find_roots.c
  src/polyfits.c
  src/cint1e_a.c src/cint3c1e_a.c
  src/cint1e_grids.c src/g1e_grids.c
  src/autocode/breit1.c src/autocode/dkb.c src/autocode/gaunt1.c
  src/autocode/grad1.c src/autocode/grad2.c src/autocode/hess.c
  src/autocode/int3c1e.c src/autocode/int3c2e.c src/autocode/intor1.c
  src/autocode/intor2.c src/autocode/intor3.c src/autocode/intor4.c
  src/autocode/int1e_grids1.c)

#option(WITH_RANGE_COULOMB "Range separated Coulomb" on)
if(WITH_RANGE_COULOMB)
# defined in cint_config.h
#  add_definitions(-DWITH_RANGE_COULOMB)
#  message("Enabled WITH_RANGE_COULOMB")
endif(WITH_RANGE_COULOMB)

if(WITH_POLYNOMIAL_FIT)
  set(cintSrc ${cintSrc} src/sr_rys_polyfits.c)
  add_definitions(-DWITH_POLYNOMIAL_FIT)
  message("Enabled WITH_POLYNOMIAL_FIT")
endif(WITH_POLYNOMIAL_FIT)

if(WITH_COULOMB_ERF)
  message("WITH_COULOMB_ERF is deprecated since v6.0")
endif(WITH_COULOMB_ERF)

if(WITH_F12)
  set(cintSrc ${cintSrc} src/g2e_f12.c src/stg_roots.c src/cint2e_f12.c)
  add_definitions(-DWITH_F12)
  message("Enabled WITH_F12")
endif(WITH_F12)

if(PYPZPX)
  add_definitions(-DPYPZPX)
  message("P orbitals convention (py, pz, px)")
else()
  message("P orbitals convention (px, py, pz)")
endif(PYPZPX)

if(MIN_EXPCUTOFF)
  message("Minimal cutoff in exponent for integral screening ${MIN_EXPCUTOFF}")
  add_definitions(-DMIN_EXPCUTOFF=${MIN_EXPCUTOFF})
endif()

if(KEEP_GOING)
  message("Do not trigger hard exit for numerical issues in Rys quadrature")
  add_definitions(-DKEEP_GOING)
endif()

option(WITH_FORTRAN "Fortran interface" on)
if(WITH_FORTRAN)
  add_definitions(-DWITH_FORTRAN)
  message("Enable Fortran interface")
else()
  message("Exclude Fortran interface")
endif(WITH_FORTRAN)

option(WITH_CINT2_INTERFACE "Old libcint (version 2) interface" on)
if(WITH_CINT2_INTERFACE)
  add_definitions(-DWITH_CINT2_INTERFACE)
  message("Enable old cint (version 2) interface")
else()
  message("Exclude old cint (version 2) interface")
endif(WITH_CINT2_INTERFACE)

option(BUILD_SHARED_LIBS "build shared libraries" 1)
option(ENABLE_STATIC "Enforce static library build" 0)
if(ENABLE_STATIC)
  set(BUILD_SHARED_LIBS 0)
endif()

add_library(cint ${cintSrc})

target_include_directories(cint
  PUBLIC
  $<BUILD_INTERFACE:${PROJECT_BINARY_DIR}/include>
  $<INSTALL_INTERFACE:${CMAKE_INSTALL_INCLUDEDIR}>
  PRIVATE
  ${PROJECT_BINARY_DIR}/src
  ${PROJECT_SOURCE_DIR}/src
)

set_target_properties(cint PROPERTIES
  C_STANDARD 99
  VERSION ${qcint_VERSION}
  SOVERSION ${cint_SOVERSION}
  LIBRARY_OUTPUT_DIRECTORY ${PROJECT_BINARY_DIR})
if(QUADMATH_FOUND)
  target_link_libraries(cint quadmath)
endif()
target_link_libraries(cint "-lm")

set(CintHeaders
  ${PROJECT_SOURCE_DIR}/include/cint_funcs.h
  ${PROJECT_BINARY_DIR}/include/cint.h)

install(TARGETS cint COMPONENT "lib")
install(FILES ${CintHeaders} DESTINATION ${CMAKE_INSTALL_INCLUDEDIR} COMPONENT "dev")
