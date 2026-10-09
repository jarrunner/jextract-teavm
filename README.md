# jextract-teavm

TeaVM `@Import` bindings generated from real C headers, using jextract's own libclang front end. Its main user is [jr](https://github.com/jarrunner/jr), which generates its Win32 and POSIX bindings with it (`jr/bindings/windows/gen-bindings.cmd`, `jr/bindings/posix/gen-bindings-posix.sh`).

You list names. The tool asks clang what each name is (function, constant, struct) and writes what the headers say. It never asks you for a type, a width, an offset or a value, so there is no place to type a wrong one.

```
jextract-teavm.cmd myapp.h --symbols myapp.symbols -I <llvm-mingw>\include ^
    --package com.example.myapp --api-class WinApi --structs-class WinOffsets -o gen --verify-c verify-win.c
```

What comes out:
- Functions become `@Import(name = "X") public static native ...`. Pointers map to `Address`, and integers map by their width on the target (`DWORD` to `int`, `WPARAM` to `long`, `INTERNET_PORT` to `short`). The C prototype and `file:line` go in the javadoc.
- Constants use clang's evaluated value, typed by width. Pointer-typed macros like `INVALID_HANDLE_VALUE` become `Address.fromLong(..)`. Wide `L"..."` macros come out correctly: jextract itself truncates these, and `WideStrings` recovers them by asking clang for each code unit.
- Structs become `SIZE` plus field offsets, plus typed accessors (`STARTUPINFOA.dwFlags(addr, v)`), so a wrong-width read cannot be written.
- `--verify-c` writes every size, offset, width and value back out as C `_Static_assert`s. Compile that file with a *different* compiler for the same target and headers (`clang --target=<t> -fsyntax-only`). A disagreement fails the build and names the symbol.
- `--include HEADER` puts `@Include(HEADER)` on the api class. TeaVM then writes `#include "HEADER"` at the top of every C file that calls one of its functions, so each call is prototyped by the real header with no forced includes and no patching of TeaVM's output; give the C compiler `-I` for the header's folder. Write `"<name.h>"` for a system header. TeaVM takes one `@Include` per class, so name a header that includes the rest (usually the one passed as HEADER above). Avoid a name a Java class also has: on a case-insensitive filesystem `jr.h` finds TeaVM's own `Jr.h`. The header needs `#pragma once`, because TeaVM's single `all.c` includes it once per calling file.

## Symbol file
One C name per line, `#` comments, optionally followed by the Java name: `SendMessageA sendMessageA`.
- `struct:stat stat_t` asks for the struct tag when an ordinary name shadows it (`stat` is also a function).
- A variadic function binds its fixed parameters, plus any Java types listed after the Java name: `open open3 int`. This is sound because TeaVM emits a plain C call compiled against the real prototype, so the C compiler performs the variadic convention itself. FFM cannot do this.
- A pointer-to-function typedef (`WNDENUMPROC`, `WNDPROC`) becomes a callback type (see below). Where a platform names no typedef, `qsort(4) QsortCompar` takes the type of a function's parameter, counted from 1.
- A function-like macro is `macro:NAME javaName (argTypes)`, for example `macro:MAKELONG makeLong (int, int)`. See below.
- `--symbols` may be repeated, so a platform can add lines of its own (a shared `posix.symbols` plus `posix-linux.symbols` or `posix-macos.symbols`).
- A name that cannot be bound is reported as `NOT BOUND: <name>: <why>` and the exit code is 1. It is never silently skipped. Reasons include a struct passed by value, a macro that needs an lvalue, or a name that is not declared anywhere. 13 such refusals were checked by a negative test while the tool was built.

## Callbacks: C calling Java
A callback type comes out as a nested abstract class in the API class, and its `invoke` is typed from the prototype exactly as an `@Import` is:
```java
public static abstract class WNDENUMPROC extends Function {
    public abstract int invoke(Address arg0, long arg1);
}
```
Get a C function pointer to a static Java method with that signature, and pass it where the C function takes the callback. Parameters of an `@Import` that take a function pointer stay `Address`:
```java
var proc = (Address) (Object) Function.get(WNDENUMPROC.class, MyClass.class, "onWindow");
enumWindows(proc, 0L);
static int onWindow(Address hwnd, long lParam) { ... }
```
- TeaVM checks the named method against `invoke` at build time and fails the build on a mismatch, naming both signatures. So the widths go header, then `invoke`, then your method, with no step typed by hand.
- **Compile with `-Wno-error=incompatible-function-pointer-types`.** TeaVM passes the Java method's own C function (`&meth_..._onWindow`), typed with `void*` where the typedef says `HWND`. The ABI is the same. clang 16 and later makes the mismatch an error by default, and with this flag it is a warning.
- A callback runs on TeaVM's own thread, and a GC inside it is safe (measured while the tool was built: 2.9 GB allocated inside 740 callbacks, and a held array survives). A callback from a thread TeaVM did not start (a thread pool, WinHTTP async) is not supported: TeaVM's runtime is single-threaded.
- `--verify-c` asserts the width of every non-pointer argument and the return type.
- jextract hands a callback's argument types over without their typedef names and without parameter names, so the javadoc shows the header's own declaration and the Java parameters are `arg0`, `arg1`. The types are exact.

## Function-like macros
libclang gives a macro no prototype, and jextract leaves function-like macros out of its model, so the symbols line gives the argument types. TeaVM emits an ordinary C call and the C preprocessor expands the macro there.
- The result type is asked of clang. A probe parse evaluates `sizeof(M(args))` and `__builtin_classify_type(M(args))`. A result type written on the line (`macro:LOWORD loword short(int)`) is checked against clang's, and a disagreement is refused.
- Argument types: `byte short char int long float double Address`.
- Refused with the reason: a name that is not a macro (a typo), an object-like macro, the wrong number of arguments, a statement-like macro (`do {} while (0)`), a struct result.
- **A macro that takes its argument's address is refused.** macOS's `WEXITSTATUS` is `(*(int *)&(x) >> 8)`. TeaVM may pass a constant (it emits `LOWORD(INT32_C(572657937))`), and `&` of a constant does not compile. Wrap such a macro in a function in a header the C build includes, and bind the function. The function must not be `static`, because jextract skips internal linkage. jr's `jr/bindings/posix/jr-posix.h` does this for `jx_wexitstatus`.
- `--verify-c` asserts the result width and type class. That also fails when the macro does not exist or takes another number of arguments.

## Structs by value
TeaVM cannot pass or return a struct by value through an `@Import`. A `org.teavm.interop.Structure` is always a `void*` in its C. Tested with `WindowFromPoint(POINT)` and `div()` returning `div_t`: clang rejects both calls. Such functions stay `NOT BOUND`.
- The way through is a one-line macro taking pointers, bound with `macro:`: `#define JX_div(n, d, out) ((void)(*(div_t *)(out) = div((n), (d))))`. Tested to pass and return by value correctly. jr carries two such shims in `jr/bindings/windows/jr-shims.h`. The shim header is added to the C compile with `-include`.

## Safer bindings: types, String overloads, ownership (opt-in)
Five options make the generated code carry what a compile-time checker needs (jr pairs them with [teavm-native-check](https://github.com/jarrunner/teavm-native-check); any other checker can read the same annotations). Every annotation and helper name is the caller's, so nothing here is specific to jr. jr's settings are in `jr/bindings/windows/gen-bindings.cmd` in the jr repository.
- `--ctype-annotation NAME`: `@NAME("...")` on every pointer whose target is known, on parameters, returns, struct classes and accessors: `"struct _FILETIME"`, `"union _LARGE_INTEGER"`, `"char"` or `"wchar_t"` for text, `"int16"` to `"int64"` for an integer by width, `"pointer"` for a pointer to a pointer. `void*` and `unsigned char*` (byte buffers) stay untyped. Wide text is known by the headers' own names (`wchar_t`, `WCHAR`, `*WSTR`, `*WCH`), because in C on Windows `wchar_t` is `unsigned short`.
- `--text-converter char=FN --text-converter wchar_t=FN --text-scope ENTER,EXIT`: for each function taking read-only text, an overload taking `String` that converts with FN, calls, and frees with EXIT(ENTER()). "Read-only" is asked of clang itself in the probe parse, so `LPCWSTR` gets an overload and `LPWSTR` (an output buffer) does not. No overload when the function returns a pointer to characters, integers or pointers (`strchr`, `PathFindFileNameW`), since that could point into the freed copy. No try/finally: native calls cannot throw, and TeaVM builds try/finally on setjmp (about 400 bytes per method).
- `--buf-class NAME`: every struct accessor also takes NAME, a bounds-checked buffer type with `getX(int)` / `putX(int, x)` and `from(int)`.
- `--returned-annotation NAME`: on the struct parameter of each field-address accessor, whose result points into that parameter.
- `--escapes-annotation NAME`, with `escapes=N[,M]` after a function's Java name in the symbols file: the C function keeps parameter N (1-based) after it returns (`atexit`, `putenv`). No header says this, so the symbols file does. Such a parameter is never converted in a String overload.

## Setup (once)
1. The official jextract build is a self-contained runtime image, and it bundles libclang: https://jdk.java.net/jextract/ (Windows x64, based on JDK 25). Unpack it to `C:\user\Apps\jextract\jextract-25`, or set `JEXTRACT_HOME`.
2. jextract is not on Maven Central. To compile this project, lift its module out of the image and install it locally:
   `jimage extract --dir img %JEXTRACT_HOME%\runtime\lib\modules`, then `jar cf org.openjdk.jextract-25.2.jar -C img\org.openjdk.jextract .`, then `mvn install:install-file -Dfile=org.openjdk.jextract-25.2.jar -DgroupId=org.openjdk -DartifactId=jextract -Dversion=25.2 -Dpackaging=jar`.
3. Run `mvn package`. `jextract-teavm.cmd` runs the shaded jar on jextract's own runtime. The layout facts are read through `--add-opens`, because jextract keeps them in package-private attribute records.

## Other platforms, from one machine
Generate for any target from one machine with `--target <triple>` and that target's headers, for example the libc headers Zig ships for cross-compiling (`zig-*/lib/libc/include`), then compile the `--verify-c` file with clang for the same target. On the same symbols the platforms really differ: `O_CREAT` 64 against 512, `st_size` at offset 48 against 96, and `mode_t` is `int` on Linux but `short` on macOS, which is exactly what this tool exists to get right.

## Folders
- `src/`: the generator.
- `upstream-fix/`: a patch to jextract itself (wide-string macros, their charset, javadoc escaping) with its standalone test, to be offered upstream.
