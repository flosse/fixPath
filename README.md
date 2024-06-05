# WIP

WARNING: `while the lld patch is working` and the concept in general works great i reimplemented the fixPath client with various
implementations like `goblin`.

expect more to come soon.

# fixPath

`fixPath` is a tool to modify the search locations for DLLs (Dynamic Shared Objects) for [Microsoft Windows 
Executables](https://learn.microsoft.com/en-us/windows/win32/debug/pe-format) by rewriting parts of the binary's PE 
header when the `.fixPath` section is present and indicates support for such rewrite _without_ having to realign 
the PE headers.

fix as in:

> fix - fasten (something) securely in a particular place or position.

List all imports:

    $ fixPath.exe --list-imports
     - (foo.dll)
     - (bar.dll) -> c:\foo\bar.dll

Change the `foo.dll` import location to an absolute path:

    $ fixPath.exe --set-import foo.dll c:\test\foo.dll

Use `fixPath` to modify the library search path:
* **[Microsoft's linker default search path(s)](https://learn.microsoft.com/en-us/windows/win32/dlls/dynamic-link-library-search-order).**, i.e. c:\Windows\System32 and similar
* **relative path** to the executable: lib\foo.dll
* **absoluate path** in filesystem: c:\lib\foo.dll

Note: `fixPath` preserves the original library name and appends this information to the `.idata` section of the PE header.

## fixPath requirements

`fixPath` requires special support from the linker `lld` - **the LLVM linker**. It can't be applied to already existing binaries.

## How does `fixPath` work?

The `fixPath` tool relies on a custom PE extension section called `.fixPath` (a section like `.idata` or `.didata`).

A patched version of the **ldd linker** or **ld linker** will create this section: 
* when passing `/useFixPath` on the command line the `.fixPath` section is created
* when passing the `/fixPathSize:33` on the command line, the `fixPathSize` can be set

The `fixPath` tool will check if the `.fixPath` field exists and if so one can use it to change the loader order from:

* The **library** filename, for instance `KERNEL32.dll` will be searched in the default locations, like `C:\Windows\System32`.
* However, it can be a **relative filepath** like `..\foo.dll` or
* It can be an **absolute filepath** like `c:\bar.dll`

### .fixPath v1

This `.fixPath` section contains:
* a `version field` [u32], `fixPathSize` [u32], empty spacer [u64] 
* `idataNameTable_size` [u32]
* array [`idataNameTable dllname`] [ascii]
* bytes padding [u32]
* `didataNameTable_size` [u32]
* array [`didataNameTable dllname`] [ascii],
* bytes padding [u32]
* URL of fixPath https://github.com/nixcloud/fixPath
* bytes padding [u32]

## LLM (LLVM compiler) support

See https://github.com/qknight/llvm-project/tree/libnix_PE-fixPath

Features:

* [x] Reserving 300 chars dllname space
* [x] Adding `.fixPath` section with **version** and **dllname size**
* [x] Adding `.fixPath` section with **dllnames** for dll loading
* [x] Adding `.fixPath` section with **dllnames** for delayed dll loading 
* [x] Using llm options to represent defaults
* [x] Command line switches for cmake & similar
* [x] Command line switches for clang-cl

### CMakeLists.txt

    set (CMAKE_CPP_FLAGS "-Xlinker /useFixPath -Xlinker /fixPathSize:333")
    set (CMAKE_C_FLAGS "-Xlinker /useFixPath -Xlinker /fixPathSize:333")

## Other linkers 

* [ ] GNU LD (GNU compiler) support
* [ ] GNU gold (GNU compiler) support
* [ ] [mold](https://github.com/rui314/mold) support
* [ ] Visual Studio linker support

No prototypes yet.

# Tests

The `tests/` directory contains a set of executables which were used for verifying the fixPath execution.

```
(base) PS C:\Users\joschie\Desktop\Projects\binutils-ld-experiments\build> cmake -G "Ninja" -D_CMAKE_TOOLCHAIN_PREFIX=llvm- .. && ninja --verbose && ./test_mylib.exe && objdump.exe -h .\test_mylib.exe
-- The C compiler identification is Clang 18.1.6 with GNU-like command-line
-- The CXX compiler identification is Clang 18.1.6 with GNU-like command-line
-- Detecting C compiler ABI info
-- Detecting C compiler ABI info - done
-- Check for working C compiler: C:/Program Files/LLVM/bin/clang.exe - skipped
-- Detecting C compile features
-- Detecting C compile features - done
-- Detecting CXX compiler ABI info
-- Detecting CXX compiler ABI info - done
-- Check for working CXX compiler: C:/Program Files/LLVM/bin/clang++.exe - skipped
-- Detecting CXX compile features
-- Detecting CXX compile features - done
-- Configuring done
-- Generating done
-- Build files have been written to: C:/Users/joschie/Desktop/Projects/binutils-ld-experiments/build
[1/4] C:\PROGRA~1\LLVM\bin\clang.exe  -IC:/Users/joschie/Desktop/Projects/binutils-ld-experiments/include -Xlinker /useFixPath -Xlinker /fixPathSize:333 -g -Xclang -gcodeview -O0 -D_DEBUG -D_DLL -D_MT -Xclang --dependent-lib=msvcrtd -MD -MT CMakeFiles/test_mylib.dir/src/main.c.obj -MF CMakeFiles\test_mylib.dir\src\main.c.obj.d -o CMakeFiles/test_mylib.dir/src/main.c.obj -c C:/Users/joschie/Desktop/Projects/binutils-ld-experiments/src/main.c
clang: warning: -Xlinker /useFixPath: 'linker' input unused [-Wunused-command-line-argument]
clang: warning: -Xlinker /fixPathSize:333: 'linker' input unused [-Wunused-command-line-argument]
[2/4] C:\PROGRA~1\LLVM\bin\clang.exe -DMYLIB_EXPORTS -Dmylib_EXPORTS -IC:/Users/joschie/Desktop/Projects/binutils-ld-experiments/include -Xlinker /useFixPath -Xlinker /fixPathSize:333 -g -Xclang -gcodeview -O0 -D_DEBUG -D_DLL -D_MT -Xclang --dependent-lib=msvcrtd -MD -MT CMakeFiles/mylib.dir/src/mylib.c.obj -MF CMakeFiles\mylib.dir\src\mylib.c.obj.d -o CMakeFiles/mylib.dir/src/mylib.c.obj -c C:/Users/joschie/Desktop/Projects/binutils-ld-experiments/src/mylib.c
clang: warning: -Xlinker /useFixPath: 'linker' input unused [-Wunused-command-line-argument]
clang: warning: -Xlinker /fixPathSize:333: 'linker' input unused [-Wunused-command-line-argument]
[3/4] cmd.exe /C "cd . && C:\PROGRA~1\LLVM\bin\clang.exe -fuse-ld=lld-link -nostartfiles -nostdlib -Xlinker /useFixPath -Xlinker /fixPathSize:333 -g -Xclang -gcodeview -O0 -D_DEBUG -D_DLL -D_MT -Xclang --dependent-lib=msvcrtd   -shared -o mylib.dll  -Xlinker /MANIFEST:EMBED -Xlinker /implib:mylib.lib -Xlinker /pdb:mylib.pdb -Xlinker /version:1.0 CMakeFiles/mylib.dir/src/mylib.c.obj  -lkernel32 -luser32 -lgdi32 -lwinspool -lshell32 -lole32 -loleaut32 -luuid -lcomdlg32 -ladvapi32 -loldnames  && cd ."
[4/4] cmd.exe /C "cd . && C:\PROGRA~1\LLVM\bin\clang.exe -fuse-ld=lld-link -nostartfiles -nostdlib -Xlinker /useFixPath -Xlinker /fixPathSize:333 -g -Xclang -gcodeview -O0 -D_DEBUG -D_DLL -D_MT -Xclang --dependent-lib=msvcrtd -Xlinker /subsystem:console CMakeFiles/test_mylib.dir/src/main.c.obj -o test_mylib.exe -Xlinker /MANIFEST:EMBED -Xlinker /implib:test_mylib.lib -Xlinker /pdb:test_mylib.pdb -Xlinker /version:0.0   mylib.lib  -lkernel32 -luser32 -lgdi32 -lwinspool -lshell32 -lole32 -loleaut32 -luuid -lcomdlg32 -ladvapi32 -loldnames  && cd ."
Hello from my_function!

.\test_mylib.exe:     file format pei-x86-64

Sections:
Idx Name          Size      VMA               LMA               File off  Algn
0 .text         00002366  0000000140001000  0000000140001000  00000400  2**4
CONTENTS, ALLOC, LOAD, READONLY, CODE
1 .rdata        00001334  0000000140004000  0000000140004000  00002800  2**4
CONTENTS, ALLOC, LOAD, READONLY, DATA
2 .data         00000200  0000000140006000  0000000140006000  00003c00  2**4
CONTENTS, ALLOC, LOAD, DATA
3 .pdata        0000039c  0000000140007000  0000000140007000  00003e00  2**2
CONTENTS, ALLOC, LOAD, READONLY, DATA
4 .fixPath      0000007b  0000000140008000  0000000140008000  00004200  2**2
CONTENTS, ALLOC, LOAD, DATA
5 .rsrc         000001a8  0000000140009000  0000000140009000  00004400  2**2
CONTENTS, ALLOC, LOAD, READONLY, DATA
6 .reloc        00000030  000000014000a000  000000014000a000  00004600  2**2
CONTENTS, ALLOC, LOAD, READONLY, DATA
```

## Research - manual editing (010 editor / ImHex)

Modifying main.exe's library entry (without extending the .rdata)

* [x] works for absolute path in main.exe
* [x] works for relative path in main.exe
* [x] works for relative path in main.exe to lib/lib.dll (will use lib/lib.dll over lib.dll in same dir as exe) 

* [x] works with long filename : "c:\t\aaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaa.dll"
* [x] works with long directory: "c:\aaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaa\t.dll" 
* [x] works with "C:\t\nix\store\zxxialnsgv0ahms5d35sivqzxqg1kicf-libiec61883-1.2.0\lib\aaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaa.dll" 213 chars
* [x] works with "C:\t\nix\store\zxxialnsgv0ahms5d35sivqzxqg1kicf-libiec61883-1.2.0\lib\aaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaa.dll" 269 chars 

# FAQ

## What motivates `fixPath`?

When compiling software using the [nix package manager](https://nixos.org), it is required to build the software from a
different folder than the final location of the software. Using `fixPath`, which is similar to the **rpath** feature
from Linux/Unix the developer can hard-code the paths where the libraries are loaded from.

It basically prevents DLL-Hell by making it possible to use absolute paths for certain libraries and not rely on
the [Microsoft's linker default search path(s)](https://learn.microsoft.com/en-us/windows/win32/dlls/dynamic-link-library-search-order).

## Why not modify any EXE file?

Modifying _dll paths of arbitrary executables_ proved way too complicated and often times resulted in corrupt
programs, because the required relocations of sections and addresses inside the sections is undocumented
and not wanted.

## rpath on Windows discussions

See:

* https://developercommunity.visualstudio.com/idea/566616/support-rpath-for-binaries-during-development.html
* https://stackoverflow.com/questions/107888/is-there-a-windows-msvc-equivalent-to-the-rpath-linker-flag

# thanks

* Martin Storsjö - <https://github.com/mstorsjo>
* John Ericson - <https://github.com/ericson2314>
