# How to Rebuild a Single File with Debug Info

When PyTorch is built with `REL_WITH_DEB_INFO`, optimizations make debugging difficult. Follow these steps to rebuild specific files with `-g -O0`.

## Steps

### 1. Get the compile command for your file

```bash
cd /pytorch/build && ninja -t compdb | grep -A2 "YourFile.cpp" | head -10
```

This outputs the full compile command. Save it.

### 2. Delete the old object file

```bash
rm /pytorch/build/<path>/CMakeFiles/<target>.dir/YourFile.cpp.o
```

Example for XPUFunctions.cpp:
```bash
rm /pytorch/build/c10/xpu/CMakeFiles/c10_xpu.dir/XPUFunctions.cpp.o
```

### 3. Compile with debug flags (bypass sccache)

Take the compile command from step 1 and:
- **Remove** `/opt/cache/bin/sccache` from the beginning (to bypass cache)
- **Replace** all `-O2` with `-g -O0`
- **Remove** `-DNDEBUG` if present

Run the modified command.

### 4. Link directly (bypass ninja)

Get the link command:
```bash
cd /pytorch/build && ninja -t compdb | grep -A2 "libc10_xpu.so" | head -10
```

Run the link command directly, or extract from the ninja compdb output:
```bash
cd /pytorch/build && /opt/cache/bin/c++ -fPIC ... -shared -Wl,-soname,libc10_xpu.so -o lib/libc10_xpu.so <all .o files> <libraries>
```

**Important:** Do NOT use `ninja <target>` - it will recompile with original flags!

### 5. Verify debug info

```bash
readelf --debug-dump=info /pytorch/build/<path>/YourFile.cpp.o 2>/dev/null | grep "DW_AT_producer" | head -1
```

Should show `-g -O0`, NOT `-O2`.

## Quick Reference: Common Targets

| File Location | Object File Path | Library |
|---------------|------------------|---------|
| `c10/xpu/*.cpp` | `c10/xpu/CMakeFiles/c10_xpu.dir/` | `lib/libc10_xpu.so` |
| `aten/src/ATen/native/*.cpp` | `caffe2/CMakeFiles/torch_cpu.dir/` | `lib/libtorch_cpu.so` |
| `torch/csrc/*.cpp` | `caffe2/torch/CMakeFiles/torch_python.dir/` | `lib/libtorch_python.so` |

## Example: XPUFunctions.cpp

```bash
# 1. Delete object file
rm /pytorch/build/c10/xpu/CMakeFiles/c10_xpu.dir/XPUFunctions.cpp.o

# 2. Compile with -g -O0 (no sccache)
cd /pytorch/build && /opt/cache/bin/c++ \
  -DHAVE_MALLOC_USABLE_SIZE=1 ... \
  -g -O0 \
  ... \
  -o c10/xpu/CMakeFiles/c10_xpu.dir/XPUFunctions.cpp.o \
  -c /pytorch/c10/xpu/XPUFunctions.cpp

# 3. Link directly
cd /pytorch/build && /opt/cache/bin/c++ -fPIC ... -shared \
  -Wl,-soname,libc10_xpu.so \
  -o lib/libc10_xpu.so \
  c10/xpu/CMakeFiles/c10_xpu.dir/*.cpp.o \
  lib/libc10.so /opt/intel/oneapi/compiler/2025.3/lib/libsycl.so

# 4. Verify
readelf --debug-dump=info /pytorch/build/c10/xpu/CMakeFiles/c10_xpu.dir/XPUFunctions.cpp.o 2>/dev/null | grep "DW_AT_producer"
```

## Notes

- The built-in `tools/build_with_debinfo.py` only works for `torch_python` target
- sccache will return cached optimized builds - always bypass it
- After relinking, restart your debug session for changes to take effect
