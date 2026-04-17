# Reproduction

## Build with ASAN

Add the following to the CMake invocation:

```
cmake -DCMAKE_BUILD_TYPE=Debug \
      -DCMAKE_CXX_FLAGS="-fsanitize=address -fno-omit-frame-pointer" \
      -DCMAKE_EXE_LINKER_FLAGS="-fsanitize=address" \
      -DFORTE_MODULE_CONVERT=ON \
      -DFORTE_MODULE_ARROWHEAD=ON \
      ..
```

The `convert` module is required to get a FB type with a struct-typed DI
port (`GEN_STRUCT_DEMUX`). The `arrowhead` module provides a compiled-in
struct type (`ArrowheadService`) that `GEN_STRUCT_DEMUX` can use as its
config string target.

## Run

```
./src/forte
```

FORTE listens on TCP 61499 by default.

## Expected ASAN Output

After sending the trigger sequence described in `poc/concept.md`:

```
==ERROR: AddressSanitizer: heap-use-after-free
READ of size 8 at 0x... thread T0
    #0 ... CDelegatingDataConnection<...>::isDelegating() const
    #1 ... CResource::deleteConnection(...)
    #2 ... CResource::executeMGMCommand(...)

0x... is located N bytes inside of M-byte region [0x..., 0x...)
freed by thread T0 here:
    #0 operator delete(void*)
    #1 CResource::deleteConnection(...)

previously allocated by thread T0 here:
    #0 operator new(unsigned long)
    #1 CDataConnection::establishGatheringConnection(...)
```

## Confirming the Fix

Apply `patch/fix.diff` and rebuild. The same trigger sequence must
complete without ASAN error. Verify the DI slot is correctly nulled
by checking that a subsequent `CreateConnection` to the same port
succeeds and installs a fresh connection without reading freed memory.
