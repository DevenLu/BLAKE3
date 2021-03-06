This is the C implementation of BLAKE3. The public API consists of one
struct and five functions in [`blake3.h`](blake3.h):

- `typedef struct {...} blake3_hasher`: An incremental BLAKE3 hashing
  state, which can accept any number of updates.
- `blake3_hasher_init(...)`: Initialize a `blake3_hasher` in the default
  hashing mode.
- `blake3_hasher_init_keyed(...)`: Initialize a `blake3_hasher` in the
  keyed hashing mode, which accepts a 256-bit key.
- `blake3_hasher_init_derive_key(...)`: Initialize a `blake3_hasher` in
  the key derivation mode, which accepts a context string of any length.
  In this mode, the key material is given as input after initialization.
  The context string should be hardcoded, globally unique, and
  application-specific. A good default format for such strings is
  `"[application] [commit timestamp] [purpose]"`, e.g., `"example.com
  2019-12-25 16:18:03 session tokens v1"`.
- `blake3_hasher_update(...)`: Add input to the hasher. This can be
  called any number of times.
- `blake3_hasher_finalize(...)`: Finalize the hasher and emit an output
  of any length. This does not modify the hasher itself. It is possible
  to finalize again after adding more input.

## Example

```c
#include <stdio.h>
#include "blake3.h"

int main() {
    // Initialize the hasher.
    blake3_hasher hasher;
    blake3_hasher_init(&hasher);

    // Write some input bytes.
    blake3_hasher_update(&hasher, "foo", 3);
    blake3_hasher_update(&hasher, "bar", 3);
    blake3_hasher_update(&hasher, "baz", 3);

    // Finalize the hash. BLAKE3_OUT_LEN is 32 bytes, but extended outputs are
    // also supported.
    uint8_t output[BLAKE3_OUT_LEN];
    blake3_hasher_finalize(&hasher, output, BLAKE3_OUT_LEN);

    // Print the hash as hexadecimal.
    for (size_t i = 0; i < BLAKE3_OUT_LEN; i++) {
      printf("%02x", output[i]);
    }
    printf("\n");
    return 0;
}
```

## Building

The Makefile included in this implementation is for testing. It's
expected that callers will have their own build systems. This section
describes the compilation steps that build systems (or folks compiling
by hand) should take. Note that these steps may change in future
versions of this repo.

### x86

Dynamic dispatch is enabled by default on x86. The implementation will
query the CPU at runtime to detect SIMD support, and it will use the
widest SIMD vectors the CPU supports. Each of the
instruction-set-specific implementation files needs to be compiled with
the corresponding instruction set explicitly enabled. Here's an example
of building a shared library on Linux:

```bash
gcc -c -fPIC -O3 -msse4.1 blake3_sse41.c -o blake3_sse41.o
gcc -c -fPIC -O3 -mavx2 blake3_avx2.c -o blake3_avx2.o
gcc -c -fPIC -O3 -mavx512f -mavx512vl blake3_avx512.c -o blake3_avx512.o
gcc -shared -O3 blake3.c blake3_dispatch.c blake3_portable.c \
    blake3_avx2.o blake3_avx512.o blake3_sse41.o -o libblake3.so
```

Note that building `blake3_avx512.c` requires both `-mavx512f` and
`-mavx512vl` under GCC and Clang, as shown above. Under MSVC, the single
`/arch:AVX512` flag is sufficient.

If you want to omit SIMD code on x86, you need to explicitly disable
each instruction set. Here's an example of building a shared library on
Linux with no SIMD support:

```bash
gcc -shared -O3 -DBLAKE3_NO_SSE41 -DBLAKE3_NO_AVX2 -DBLAKE3_NO_AVX512 \
    blake3.c blake3_dispatch.c blake3_portable.c -o libblake3.so
```

### ARM

TODO: add NEON support to `blake3_dispatch.c`.

### Other Platforms

The portable implementation should work on most other architectures. For
example:

```bash
gcc -shared -O3 blake3.c blake3_dispatch.c blake3_portable.c -o libblake3.so
```

Most multi-platform builds should build `blake3_sse41.c`,
`blake3_avx2.c`, and `blake3_avx512.c` when targetting x86, and skip
them for all other platforms. It could be possible to `#ifdef` out the
contents of those files on non-x86 platforms, but flags like `-msse4.1`
generally cause errors anyway when they're not supported, so skipping
these build steps entirely is usually necessary.

## Differences from the Rust Implementation

The single-threaded Rust and C implementations use the same algorithms
and have essentially the same performance if you compile with Clang.
(Both Clang and rustc are LLVM-based.) Note that performance is
currently better with Clang than with GCC.

The C implementation does not currently support multi-threading. OpenMP
support or similar might be added in the future.

Both the C and Rust implementations support output of any length, but
only the Rust implementation provides an incremental (and seekable)
output reader. This might also be added in the future.
