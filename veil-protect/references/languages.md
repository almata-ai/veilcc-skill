# Per-language integration

VeilCC works on compiled objects and LLVM IR, not source — anything that lowers
through LLVM is in reach. The one rule everywhere: a function you name at build time
must have the **C ABI** so it survives with a stable, un-mangled symbol.

## C / C++

Annotate in source, build as usual, swap the object. Auto-discovery finds marked
functions — you don't repeat the name.

```c
__attribute__((annotate("veil:seal")))   // or "veil:max" for the strongest level
uint32_t check_license(const char *key) { /* … */ }
```

```sh
clang -O2 -c license.c -o license.o
veil protect --level seal license.o -o license.sealed.o
clang -O2 license.sealed.o -o app          # link as normal
```

C++ is identical — give the function `extern "C"` for a stable name, or pass the
mangled name to `--proto`.

## Rust

No `annotate` attribute, so export with the C ABI and name the symbol + signature in
C terms.

```rust
#[no_mangle]
pub extern "C" fn check_license(key: *const u8, len: usize) -> u32 { /* … */ }
```

```sh
veil protect --level seal \
    --symbols check_license \
    --proto "uint32_t check_license(const unsigned char*, size_t)" \
    liblicense.o -o license.sealed.o
```

## C# via Unity IL2CPP

**Recommended — protect a native plugin.** Move the logic worth stealing (license
check, anti-cheat core, proprietary kernel) into a small C/C++ or Rust plugin, call
it from C# over P/Invoke, and seal the plugin. The secret never exists in managed IL
or in the IL2CPP output at all. Build it, `veil protect --level seal`, drop it in
`Assets/Plugins/`. Device-bound keys matter most here (games / anti-cheat).

**Advanced — target the IL2CPP output.** If the logic must stay in C#, have IL2CPP
emit and keep its generated C++ (`Il2CppOutput/`) and object files, then protect the
compiled objects by symbol before the final link. Caveat: IL2CPP symbol names can
shift between builds/Unity versions, so wire a name check into the build. The
native-plugin route is the verified, recommended path.

## The C-ABI rule (summary)

- **Why:** the sealer lifts the function out by symbol and the loader calls it back —
  both need a predictable name and calling convention.
- **How:** `extern "C"` (C++), `#[no_mangle] pub extern "C"` (Rust), a plain exported
  C function (plugins). Pass the signature to `--proto` in C types.
