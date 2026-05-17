# pgvector on Android ARM64 (Termux)

PostgreSQL's `vector` extension (pgvector) has no pre-built package for Termux. It must be compiled from source with a specific linker fix.

## Compilation

```bash
# Install build tools (usually present)
pkg install -y cmake ninja
```

These must be installed **before** any `pip install` that triggers cmake — pip tries to build cmake from source if not found, which fails.

### Standard Build (fails at runtime)

```bash
git clone --depth 1 https://github.com/pgvector/pgvector.git
cd pgvector
make
make install
```

This produces `vector.so` but **fails at runtime** with:
```
ERROR: could not load library "vector.so": dlopen failed: 
cannot locate symbol "acos" referenced by "vector.so"
```

### Fix: Re-link with `-lm`

The `acos` function is in `libm` on Android, but pgvector's Makefile doesn't link it explicitly since desktop Linux bundles it differently.

```bash
cd pgvector
# Rebuild .o files
make clean && make

# Re-link shared library with -lm appended
aarch64-linux-android-clang -shared -o vector.so src/*.o \
  -L/data/data/com.termux/files/usr/lib \
  -Wl,-rpath=/data/data/com.termux/files/usr/lib \
  -lm -fvisibility=hidden

# Install the fixed .so
cp vector.so /data/data/com.termux/files/usr/lib/postgresql/vector.so
```

Or use the `SHLIB_LINK` make override (one-liner):
```bash
SHLIB_LINK="-lm -L/data/data/com.termux/files/usr/lib -Wl,-rpath=/data/data/com.termux/files/usr/lib" make
```

### Verify

```bash
psql -d yourdb -c "CREATE EXTENSION IF NOT EXISTS vector;"
# Should return CREATE EXTENSION — no errors.
```

## Why This Happens

On standard Linux, `libm` is a separate shared library that the linker resolves at build time. On Android (Bionic libc), `libm` exists as a separate `.so` but the dynamic linker doesn't automatically resolve `acos` unless `-lm` is explicitly passed during linking. Desktop Linux's build toolchain adds `-lm` implicitly; Android's doesn't.
