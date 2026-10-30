qcint (quick libcint) 
======================

An optimized libcint branch for X86 platform

version 6.2.0
2026-10-16


What is qcint
-------------
(GPL licensed) qcint is a branch of [libcint](https://github.com/sunqm/libcint.git)
library.  It provides exactly the same APIs as libcint.  However, the code
is optimized against SSE3, AVX, AVX2 and AVX-512 instructions.  On x86_64 platform, qcint can
be 5 ~ 50% faster than libcint.  Please refer to libcint for
more details of the features and installation instructions.

Unlike libcint, qcint does not support the integer-8 build (`-DI8=1`): all
integers in the API are 32-bit `int`, and configuring with `-DI8=1` fails.
`-DCACHE_SIZE_I8=1` is supported; it makes `CACHE_SIZE_T`, the return type
of the integral functions (which return the required cache size when `out` is
NULL), `int64_t`.


Bug report
----------
Qiming Sun <osirpt.sun@gmail.com>

