# C foreign function interface

## Summary

This proposes a C foreign function interface for Lute. A binding loads symbols from a shared library or the running process, describes the header's types as Luau values, and calls through libffi. `ffi` holds libraries and `ffi.c` holds types, values and memory.

```luau
local ffi = require("@lute/ffi")
local c = ffi.c

-- typedef struct { int quot; int rem; } div_t;
local divT = c.struct({
	{ quot = c.int },
	{ rem = c.int },
}, "div_t")

-- div_t div(int numer, int denom);
local div = ffi.process:load("div", c.func(divT, c.int, c.int))

local d = div(17, 5)
print(d.quot:asNumber(), d.rem:asNumber()) -- 3 2
```

## Motivation

Lute reaches native code only through modules compiled into the runtime. A new binding needs a change to Lute and a new release, and Lute maintains its build from then on. A helper program through `process.run` returns only after the child exits, and state such as an open database does not survive between calls.

Users have asked to reach native libraries in [#191](https://github.com/luau-lang/lute/issues/191), [#1047](https://github.com/luau-lang/lute/issues/1047), [#1185](https://github.com/luau-lang/lute/issues/1185) and [#1212](https://github.com/luau-lang/lute/issues/1212), for graphics bindings, system APIs and Rust code. [#968](https://github.com/luau-lang/lute/pull/968) proposed loading compiled modules through `require` and was closed in favor of exploring an FFI. The FFI scaffolds are merged into `ffi` ([#356](https://github.com/luau-lang/lute/pull/356)) and `green/ffi` ([#1012](https://github.com/luau-lang/lute/pull/1012)). An FFI lets a package bind a C library in Luau, as `Deno.dlopen`, `bun:ffi`, LuaJIT's `ffi` and Python's `ctypes` do.

The hard part is lifetime, because C libraries keep pointers after a call returns. `sqlite3_bind_text` with `SQLITE_STATIC` requires the text to stay valid until the statement is finalized or the parameter is rebound. LuaJIT leaves anchoring to the caller, even for a pointer derived by an offset. This design anchors the views and pointers the runtime derives. What C keeps, a binding attaches to the handle C keeps it in, and the runtime frees it only after that handle's finalizer has run.

## Design

The interface has four public types. A `CType` describes a C type. A `CData` is a value of a `CType` in native memory, and a `CAddress` is an address read from one. A `LibraryHandle` is a shared library or the running process, and loading a symbol from it gives a `CData`. `ffi` holds library handles, and `ffi.c`, written `c` below, holds the functions that build and take the other types.

The [interface](#interface) assembles the two public tables. The subsections after it define [types](#types), [values](#values), [libraries](#libraries) and [calls](#calls) in that order. Two sections then hold the rules that span types: [conversions](#conversions) and [ownership and lifetime](#ownership-and-lifetime).

The type definitions shown here are simplified. The strict definitions also carry field, element and const types and catch many initializer mistakes, with the use of type functions. Arguments are checked at run time.

### Interface

The `c` table is one type assembled from a block per role. Each block is defined in the section where its rules live, and the list below links to it. The `ffi` table is an `FFILibrary`.

```luau
--- The type of `ffi.c`, written `c` in this document.
export type CLibrary = CPrimitives & CTypeConstructors & CMemory & CLifetime & CErrno
```

- `CPrimitives` holds the primitive types, and `CTypeConstructors` the functions that build the other types, under [types](#types).
- `CMemory` holds the functions that allocate, inspect, cast and compare values, under [values](#values).
- `CLifetime` holds `c.gc`, `c.attach` and `c.detach`, under [ownership and lifetime](#ownership-and-lifetime).
- `CErrno` holds `c.errno` and `c.lastError`, under [errno and last error](#errno-and-last-error).
- `FFILibrary` holds `ffi.open`, `ffi.process`, `ffi.pointerSize` and `ffi.c`, under [libraries](#libraries).

### Types

A `CType` describes a C type. The runtime computes struct layout and the calling convention from it, which a binding would otherwise hand-code per platform. This section gives the definition, the primitives, the constructors and type equality. The capture methods of function types are under [errno and last error](#errno-and-last-error).

```luau
--- A C type.
export type CType = {
	read tag: "void" | "primitive" | "pointer" | "array" | "struct" | "union" | "function" | "opaque",
	--- In bytes.
	read size: number,
	read align: number,
	read isConst: boolean,

	ptr: (self: CType) -> CType,
	array: (self: CType, length: number) -> CType,
	const: (self: CType) -> CType,
}

--- The fields each tag adds.
export type CPrimitiveType = CType & { read tag: "primitive", read primitive: string }
export type CPointerType = CType & { read tag: "pointer", read pointee: CType }
export type CArrayType = CType & { read tag: "array", read elementType: CType, read length: number }
export type CStructType = CType & { read tag: "struct", read name: string?, read fields: { CStructFieldLayout } }
export type CUnionType = CType & { read tag: "union", read name: string?, read members: { CUnionMemberLayout } }
export type COpaqueType = CType & { read tag: "opaque", read name: string }
export type CFunctionType = CType & {
	read tag: "function",
	read result: CType,
	read params: { CType },
	captureErrno: (self: CFunctionType) -> CFunctionType,
	captureLastError: (self: CFunctionType) -> CFunctionType,
}

export type CStructFieldLayout = { read name: string, read type: CType, read offset: number, read align: number }
export type CUnionMemberLayout = { read name: string, read type: CType, read align: number }

--- The primitive types of `c`.
export type CPrimitives = {
	read void: CType & { read tag: "void" }, read voidptr: CPointerType,
	read bool: CPrimitiveType, read char: CPrimitiveType, read schar: CPrimitiveType, read uchar: CPrimitiveType,
	read short: CPrimitiveType, read ushort: CPrimitiveType, read int: CPrimitiveType, read uint: CPrimitiveType,
	read long: CPrimitiveType, read ulong: CPrimitiveType, read float: CPrimitiveType, read double: CPrimitiveType,
	-- Fixed width integers.
	read int8_t: CPrimitiveType, read int16_t: CPrimitiveType, read int32_t: CPrimitiveType, read int64_t: CPrimitiveType,
	read uint8_t: CPrimitiveType, read uint16_t: CPrimitiveType, read uint32_t: CPrimitiveType, read uint64_t: CPrimitiveType,
	-- Sized by the platform.
	read size_t: CPrimitiveType, read ssize_t: CPrimitiveType, read intptr_t: CPrimitiveType, read uintptr_t: CPrimitiveType,
	read wchar_t: CPrimitiveType,
}

--- The functions of `c` that build types. Fields and members are an array of single-key tables.
export type CTypeConstructors = {
	read struct: (fields: { { [string]: CType } }, name: string?) -> CStructType,
	read union: (members: { { [string]: CType } }, name: string?) -> CUnionType,
	read func: (result: CType, ...CType) -> CFunctionType,
	read opaque: (name: string) -> COpaqueType,
}
```

The primitives are the fields of `CPrimitives`. `c.voidptr` is `c.void:ptr()`. The methods build types from a type, and `CTypeConstructors` builds the rest.

`t:ptr()` is a pointer to `t`, and `t:array(n)` is an array of `n` elements of `t`. `t:const()` keeps the `tag` and fields of `t`, sets `isConst` and is not equal to `t`. `t:const():const()` is `t:const()`. As in C, `t:array(n):const()` is `t:const():array(n)`. Apart from the capture methods of function types, a `CType` has no other methods.

Struct fields and union members are an array of single-key tables, in order, such as `c.struct({ { x = c.float }, { y = c.float } })`. The optional name affects diagnostics only. `c.func` lists the result and then the parameters, in the order of a C declaration and of Python's `CFUNCTYPE`. `c.opaque` creates a type of unknown layout. Each call creates a distinct type, only pointers to it are usable, and reading its `size` or `align` raises. A type whose size does not fit in 64 bits raises when it is built.

Two `CType`s are equal when their structure is. A struct's name does not count and its field names do.

### Values

A `CData` is typed native storage, which C APIs need for structs, arrays and out-parameters. This section gives the definition and its operations, then construction, access, reading back and identity. How Luau values convert into a `CData` is under [conversions](#conversions), and what keeps its memory alive under [ownership and lifetime](#ownership-and-lifetime).

```luau
--- A value of a C type in native memory.
export type CData = {
	--- Array and pointer elements by index, struct fields and union members by name.
	read [number | string]: CData,

	asNumber: (self: CData) -> number,
	asBoolean: (self: CData) -> boolean,
	--- Copies bytes from a byte array or byte pointer.
	asString: (self: CData, length: number?) -> string,
	--- Stores `value` under the rules in "Conversions".
	write: (self: CData, value: any) -> (),
	--- An array's base address, or a pointer or function address. Other types raise.
	address: (self: CData) -> CAddress,
	--- A pointer to this storage, or to the code of a function.
	ref: (self: CData) -> CData,
	isNull: (self: CData) -> boolean,
	--- A view of the pointed-to value. Raises for a null or `void` pointer.
	deref: (self: CData) -> CData,
	--- A pointer advanced by `n` elements. Raises for a null pointer.
	offset: (self: CData, n: number) -> CData,
}

--- An opaque address. The field exists only in the type.
export type CAddress = { read __address: never }

--- The functions of `c` that produce or take a `CData`.
export type CMemory = {
	read new: (t: CType, init: any?) -> CData,
	read string: (s: string) -> CData,
	read alloc: (n: number, element: CType?) -> CData,
	read typeof: (v: CData) -> CType, read sizeof: (v: CData) -> number, read alignof: (v: CData) -> number,
	read field: (v: CData, name: string) -> CData,
	read toBuffer: (v: CData) -> buffer,
	read cast: (v: number | CData, t: CType) -> CData,
	read equals: (a: CData, b: CData) -> boolean,
	read addressToBuffer: (a: CAddress) -> buffer, read addressFromBuffer: (b: buffer) -> CAddress,
}
```

Only operations used on nearly every value are methods, since a method hides a C field of the same name. `CMemory` types the functions in `c` that produce or take a `CData`, and these rules apply to them.

- `c.new` allocates zeroed storage, then applies the initializer. It raises for `void`, function and opaque types.
- `c.string` copies a Luau string into a new null-terminated `char` array.
- `c.alloc` allocates `n` zeroed elements and returns a pointer to them. Without an element type it allocates `n` bytes as `void*`.
- `c.typeof`, `c.sizeof` and `c.alignof` inspect the value's type, and raise for a `CType`, which carries its own `size` and `align`.
- `c.field` views a field by name, including one a method hides.
- `c.toBuffer` copies the value's storage bytes.
- `c.cast` converts `v` to `t` like a C cast, under [casts](#casts).
- `c.equals` compares contents, under [equality](#equality).
- `c.gc` attaches a finalizer, under [finalizers](#finalizers). `c.attach` keeps a value alive as long as an owner and returns the id it used, and `c.detach` releases an id or a value, under [attached values](#attached-values). `CLifetime` types all three.
- `c.addressToBuffer` and `c.addressFromBuffer` expose an address's bits.

A function `CData` is called like a Luau function, under [calls](#calls).

Indexing is zero-based. Indices, offsets, lengths and counts must be integers in range. Array indexing checks the declared length, and pointer indexing does not. Indexing and `offset` raise for a pointer to `void` or to a function, which has no element size, and `p:deref()` gives the function. `#v` is an array's length and raises for other types. Reading a field, union member or element gives a view of the same storage, which assignment and `:write(value)` update.

```luau
local exponent = c.new(c.int)
-- double frexp(double x, int *exp);
local frexp = ffi.process:load("frexp", c.func(c.double, c.double, c.int:ptr()))
print(frexp(48, exponent:ref()), exponent:asNumber())

local values = c.new(c.int:array(3), { 10, 20, 30 })
values[1]:write(25)
print(values[1]:asNumber())
```

`asNumber()` and `asBoolean()` read a primitive and raise for any other type, as `deref()` and `offset(n)` do for a non-pointer. `asNumber()` is the explicit narrowing to a Luau number, so it rounds a value a double cannot hold instead of raising. It and numeric addresses can lose precision beyond 2^53. Without a length, `asString` stops at a null byte or an array's end.

Each operation returns a new `CData`, even over the same storage, so `rawequal` and table keys tell them apart. A `CAddress` has address equality and table-key identity, so a binding can key a table by a handle C returns. `v:address()` or `v:ref():address()` serves as a key for any value.

### Libraries

This section covers library handles and the `ffi` table. Calls through a loaded function are under [calls](#calls).

```luau
--- A shared library, or the symbols of the running process.
export type LibraryHandle = {
	read path: string,
	read kind: "dynamic" | "process",
	load: ((self: LibraryHandle, name: string, type: CType) -> CData)
		& ((self: LibraryHandle, types: { [string]: CType }) -> { [string]: CData }),
	isOpen: (self: LibraryHandle) -> boolean,
	close: (self: LibraryHandle) -> (),
}

--- The type of `ffi`.
export type FFILibrary = {
	read open: (nameOrPath: string) -> LibraryHandle,
	read process: LibraryHandle,
	read pointerSize: number,
	read c: CLibrary,
}
```

`ffi.open` opens a shared library, `ffi.process` exposes the running process's symbols and `ffi.pointerSize` reports pointer width in bytes.

`ffi.open` expands a plain name to the platform's shared-library filename. Paths and names containing a recognized extension are used unchanged. It raises when the library cannot be loaded.

`lib:load(name, type)` binds a function or exported data symbol, and `lib:load({ name = type })` binds several, keyed by string. Missing symbols raise. After `close`, `load` raises, and loaded symbols keep the library loaded until collected. Repeated close is harmless. `ffi.process:close()` raises.

### Calls

This section covers calls through a loaded function, then errno capture. How arguments convert is under [conversions](#conversions).

Calls require the declared argument count. A pointer argument converts under [pointers](#pointers) and [strings](#strings). Any other argument converts as a write does under [initializers and writes](#initializers-and-writes), except that `nil` raises. A struct argument therefore takes a field dictionary, a matching `CData` or a buffer. Arguments stay alive until the call returns. Calls are synchronous and block Lute's event loop.

`void` returns no value. `bool` returns a boolean. `float`, `double` and integer types of 32 bits or fewer return numbers, apart from `long`, `ulong` and the size and pointer-sized integers. Those, 64-bit integers, pointers, and structs return `CData`. This holds even where `long` is 32-bit.

`c.func` raises for an array parameter, which C declares as a pointer, and for an array or function result. It also raises for a union, or an aggregate containing one, passed or returned by value, since libffi cannot classify them. An aggregate of more than 4096 members, counting each array element, raises too, since libffi describes it member by member.

#### Errno and last error

System APIs report failure through `errno`, which the runtime can overwrite before a script reads it. Capture is therefore part of the function type.

```luau
--- The functions of `c` that read what a capturing call saved.
export type CErrno = {
	read errno: () -> number,
	read lastError: () -> number,
}
```

`f:captureErrno()` returns a function type, not equal to `f`, that captures `errno` right after every call through it returns. `c.errno()` returns the value captured by the most recent such call in the current VM, or 0 before any capture. `f:captureLastError()` and `c.lastError()` do the same for `GetLastError()` on Windows, and `c.lastError()` returns 0 elsewhere. Capture does not clear the value before the call.

The capture options are methods, as `const` is, so that `c.func` takes only C types and its parameters can be typed as one list.

### Conversions

This section covers how Luau values become C values in constructors, `:write()` and call arguments. Its parts are initializers and writes, numbers, casts, pointers, strings and equality, in that order.

#### Initializers and writes

Primitives accept numbers, booleans, or a `CData` of a matching type, meaning the same type apart from const, and reject numeric strings. Struct initializers name every field and no others. Arrays accept element tables no longer than the array. Construction zeroes omitted elements, and writes leave them unchanged. A union raises for a table, so a binding writes a member through its view.

Non-pointer values also accept a buffer of at least their size, whose first bytes they take as raw native bytes, or matching `CData` as a byte copy. Passing `nil` to `:write()` zeroes storage. Construction initializes const storage, and later writes through it raise.

#### Numbers

Implicit conversions in calls, constructors and writes raise rather than wrap, so a wrong number fails where the binding passes it. An integer target raises for a fractional value, NaN, an infinity, or a value outside its range. A `float` target rounds to nearest and raises for a finite value beyond its range. NaN and infinities pass to floating-point targets unchanged.

#### Casts

Only `c.cast` wraps, and it defines every case that C leaves undefined. A number cast to an N-bit integer type truncates toward zero, then wraps modulo 2^N. NaN and infinities cast to integers become 0. Integer casts wrap the same way. A cast to `float` rounds to nearest, and overflows to an infinity. A cast to `bool` gives true for any nonzero value, NaN included.

Pointers cast to and from other pointers, integers and `bool`, and arrays and functions cast to pointers. A pointer cast to an integer narrower than a pointer raises. Any other cast raises, including one between a pointer and a floating type.

#### Pointers

Pointer initialization, writes and arguments accept `nil`, a numeric address, `CAddress`, a buffer's data address, or `CData`. A typed pointer takes a `CData` of the same pointee or element type, and a pointer to `t:const()` also takes `t`. A `void*` takes any `CData`, and a non-pointer supplies its storage address.

#### Strings

A Luau string can be passed directly for a `const char*` parameter. C receives the string's own bytes, which stay valid until the call returns, and an embedded null ends the string as C sees it. Other pointer parameters reject Luau strings. Anything C retains after the call needs `c.string(s)`.

Byte arrays take strings and reject numbers. Writes copy the bytes without adding a terminator or clearing the tail, and a string longer than the array raises. `wchar_t` has no string conversion.

#### Equality

`==` between two `CData` compares values for the pairs below and is false for any other pair, so generic code such as `table.find` can compare any values. Primitives of one type compare by value, so NaN equals nothing and `+0` equals `-0`. Integers of different widths and one signedness compare by value too. Plain `char` and `wchar_t` take the platform's signedness. Other primitive pairs are false where C would convert one side first, such as `int` with `uint` or `float` with `double`. `bool` compares only with `bool`. Pointers, arrays and functions compare by the address they hold or decay to, when their pointee types match apart from const or one is `void`. Structs and unions are false. `==` with anything that is not a `CData` is false, so `v == 0` is false whatever `v` holds, and `p == nil` is false for a null pointer. `p:isNull()` tests for null.

`c.equals(a, b)` takes types that match apart from const and compares field by field and element by element, ignoring padding. Primitives and pointers compare as `==` does. A union inside raises.

### Ownership and lifetime

This section says which memory the runtime frees, what keeps it alive, and how a binding releases memory C hands out. Its functions are `CLifetime`, and the parts after the rules are [finalizers](#finalizers) and [attached values](#attached-values).

```luau
--- The functions of `c` that manage lifetime.
export type CLifetime = {
	read gc: (v: CData, fn: ((CData) -> ()) | CData | nil) -> CData,
	read attach: (value: CData, owner: CData, id: number?) -> number,
	read detach: (target: CData | number, owner: CData) -> (),
}
```

Memory from `c.new`, `c.alloc` and `c.string` belongs to the runtime. It counts toward the collector's pacing and is freed once no value keeps it alive. A value from `c.new` or `c.string` keeps its own memory alive. A pointer from `c.alloc` keeps the memory it allocated alive until its `:write()` stores a different address. Other values keep alive the following.

| Value | Keeps alive | So that |
| --- | --- | --- |
| A view: a field, union member or element, or `p:deref()` | The memory it was read from, and any finalizer on that memory. Through a pointer that holds a target, that is the target. Through any other pointer, it is the pointer's own memory | `s.a` stays valid after `s` is dropped, and a view through a pointer with a finalizer keeps that finalizer from running |
| A pointer from `:ref()`, `:offset(n)`, or a cast from an array, a function or a pointer | What a view through its source keeps alive, until its `:write()` stores a different address | Such a pointer passed to C stays valid while the script holds it |
| A pointer read from memory, returned by C, or built from an address | Nothing, even when it points into runtime memory | Finding that memory needs the address registry of a later milestone |

A pointer that stores a different address lets go of what it kept, so `p:write(p:offset(1))` can leave a `c.alloc` pointer `p` dangling. Assign `p = p:offset(1)` instead.

Storing a pointer into memory keeps nothing alive, and neither does passing one to C. Keeping stored pointers alive would need collector support for cycles, which [out of scope](#out-of-scope) leaves to a later milestone. As in LuaJIT, the script keeps reachable whatever C or a stored pointer still uses, in a table the binding owns or with `c.attach` under [attached values](#attached-values). `c.new(pairT, { a = 1, b = c.string("x") })` leaves `b` dangling once the string is collected, so the example below attaches the string to `t`.

A `CData` belongs to the VM that created it and cannot be passed to another VM.

#### Finalizers

C libraries hand out memory and handles that must be released through their own function, such as `free` or `sqlite3_close`. `c.gc(v, fn)` makes that call after collection, and returns `v`. `fn` may be a Luau function or a `CData` function, which must take one parameter that accepts `v` under the call rules.

Once nothing keeps `v`'s memory alive, the collector only queues `fn`. The queue runs whenever the script yields to the runtime. `fn` then receives a new value of `v`'s type holding the bytes `v` last held. It may yield, and an error in it is reported like an error in any scheduled task. Finalizers run in unspecified order, apart from the order [attached values](#attached-values) give them. Before a VM closes, the runtime runs every finalizer still queued or attached, including those of values still reachable. A script that never yields frees that memory only at exit, and `process.exit` ends the process without running them.

The runtime holds `fn` itself, so a Luau `fn` that captures `v`, as `function() free(v) end` does, keeps `v` alive and runs only when the VM closes. `c.gc(v, nil)` removes the finalizer, so `free(c.gc(p, nil))` frees early.

A finalizer belongs to the memory of that one value. A copy read back from memory carries none, and finalizers on two values over the same memory release it twice. `c.gc` raises for a function, for a loaded symbol, for a derived pointer into memory the runtime owns, and for a view. A view is a temporary, so its finalizer would release memory its parent still points at. The error says to attach the finalizer to the value the view came from, as `s` takes it below for the field `s.b`.

```luau
-- char *strdup(const char *s);
local strdup = ffi.process:load("strdup", c.func(c.char:ptr(), c.char:const():ptr()))
-- void free(void *p);
local free = ffi.process:load("free", c.func(c.void, c.voidptr))
local copy = c.gc(strdup("hello"), free)
print(copy:asString())

local pairT = c.struct({ { a = c.int }, { b = c.char:ptr() } })
local s = c.gc(c.new(pairT, { a = 1, b = strdup("owned") }), function(p) free(p.b) end)

local name = c.string("x")
local t = c.new(pairT, { a = 2, b = name })
c.attach(name, t) -- t.b stays valid as long as t
t.b:write(nil)
c.detach(name, t) -- t no longer points at name, so it stops keeping it
```

#### Attached values

C libraries keep pointers in their own memory, where the runtime cannot see them. `sqlite3_bind_text` with `SQLITE_STATIC` keeps the text until the statement is finalized or the parameter is rebound. `c.attach(value, owner, id)` makes `owner` keep `value` alive, so the statement handle keeps the text it was given. Attaching a value also keeps alive whatever that value keeps alive. `c.attach` raises for a `value` that is not a `CData`.

Each attachment has an id, an integer that belongs to `owner`. An explicit `id` is a non-negative integer, and attaching under it releases what it held, so rebinding a parameter in a loop keeps one text per parameter. C may use the old value until the call that replaces it returns, so `c.attach` comes after that call. Without `id` the runtime picks a negative id this owner has not used, so it never replaces an explicit one. `c.attach` returns the id it used in both cases, not `value`. Attaching in a loop without an id keeps every value until `owner` is freed, so pass an id whenever C replaces what a slot holds.

`c.detach(target, owner)` releases at once, so it comes after C has stopped using the value. A number `target` releases what that id holds. A `CData` `target` releases every attachment of the same value, under whatever ids it was attached. For runtime memory that means every attachment whose value sits in the same memory, so a view of the value matches it and a pointer derived from it does not. A loaded data symbol matches by its library and address. The address cannot be reused while the library stays loaded, and the attachment keeps it loaded. An id that holds nothing, or a value not attached, does nothing.

The runtime releases what is attached to `owner` once nothing keeps `owner`'s memory alive and `owner`'s finalizer, if it has one, has returned. The finalizer can therefore still use the attached values, and the finalizer of an attached value runs after `owner`'s. When a VM closes, the runtime runs an owner's finalizer to completion before the finalizers of the values attached to it. Attachments belong to `owner`'s memory as its finalizer does, so a copy read back from memory has none. For a pointer that memory is the pointer's own storage, not the object it points at, since the runtime cannot see C free an address and reuse it. Attach to the value that carries the finalizer, or to the one the script passes to the call that releases the handle.

`c.attach` raises for the owners `c.gc` raises for. It also raises when `value` already keeps `owner`'s memory alive, directly or through views, derived pointers or values attached to it, because such a cycle could never be released.

## Examples

Each example runs on its own with `ffi` and `c` from the summary, and calls libc through `ffi.process`. The first also opens zlib, which on Linux needs the unversioned `libz.so` from the development package. The C declaration is in a comment above each symbol.

### Libraries and symbols

```luau
local zlib = ffi.open("z")
-- const char *zlibVersion(void);
local zlibVersion = zlib:load("zlibVersion", c.func(c.char:const():ptr()))
print(zlibVersion():asString()) -- the installed version, such as 1.3

-- Several symbols in one call, keyed by name.
local libc = ffi.process:load({
	abs = c.func(c.int, c.int),
	strlen = c.func(c.size_t, c.char:const():ptr()),
})
print(libc.abs(-42), libc.strlen("hello"):asNumber()) -- 42 5

-- Closing stops new loads. A loaded symbol keeps the library loaded.
zlib:close()
print(zlib:isOpen(), zlibVersion():asString()) -- false 1.3
```

### Handles, errno and finalizers

```luau
local fileT = c.opaque("FILE")
local cstr = c.char:const():ptr()
-- FILE *fopen(const char *path, const char *mode);
local fopen = ffi.process:load("fopen", c.func(fileT:ptr(), cstr, cstr):captureErrno())
-- FILE *tmpfile(void);
local tmpfile = ffi.process:load("tmpfile", c.func(fileT:ptr()))
-- int fclose(FILE *f);
local fclose = ffi.process:load("fclose", c.func(c.int, fileT:ptr()))
-- int fputs(const char *s, FILE *f);
local fputs = ffi.process:load("fputs", c.func(c.int, cstr, fileT:ptr()))
-- void rewind(FILE *f);
local rewind = ffi.process:load("rewind", c.func(c.void, fileT:ptr()))
-- char *fgets(char *s, int n, FILE *f);
local fgets = ffi.process:load("fgets", c.func(c.char:ptr(), c.char:ptr(), c.int, fileT:ptr()))

-- Failure comes back as a null pointer, and errno was captured right after the call.
local missing = fopen("/no/such/file", "r")
print(missing:isNull(), c.errno()) -- true 2

-- fclose is queued once nothing keeps the handle alive, and runs when the script yields.
local file = c.gc(tmpfile(), fclose)
fputs("hello", file)
rewind(file)
local line = c.new(c.char:array(32))
fgets(line, 32, file)
print(line:asString()) -- hello
```

### Structs and views

```luau
local vec2T = c.struct({ { x = c.float }, { y = c.float } }, "Vec2")
local rectT = c.struct({ { min = vec2T }, { max = vec2T } }, "Rect")

local r = c.new(rectT, { min = { x = 0, y = 0 }, max = { x = 4, y = 3 } })
r.max.x = 5 -- assignment and r.max.x:write(5) are the same store
print(r.max.x:asNumber(), r.max.y:asNumber()) -- 5 3

-- A struct result comes back as a CData.
-- typedef struct { int quot; int rem; } div_t;
-- div_t div(int numer, int denom);
local divT = c.struct({ { quot = c.int }, { rem = c.int } }, "div_t")
local div = ffi.process:load("div", c.func(divT, c.int, c.int))
local d = div(17, 5)
print(d.quot:asNumber(), d.rem:asNumber()) -- 3 2
```

### Arrays and pointers

```luau
local arr = c.new(c.int:array(5), { 10, 20, 30, 40, 50 })
print(arr[4]:asNumber()) -- 50, and arr[5] raises because an array knows its length

-- A pointer does not, so p[5] would read past the array unchecked.
local p = c.cast(arr, c.int:ptr())
print(p[2]:asNumber(), p:offset(3):deref():asNumber()) -- 30 40

-- void *memset(void *s, int c, size_t n);
local memset = ffi.process:load("memset", c.func(c.voidptr, c.voidptr, c.int, c.size_t))
memset(arr, 0, c.sizeof(arr)) -- a void* takes any CData
print(arr[0]:asNumber()) -- 0
```

### Out-parameters

```luau
-- long strtol(const char *s, char **end, int base);
local strtol = ffi.process:load("strtol", c.func(c.long, c.char:const():ptr(), c.char:ptr():ptr(), c.int))

local text = c.string("1234abc")
local rest = c.new(c.char:ptr()) -- the cell C writes the end pointer into
local value = strtol(text, rest:ref(), 10)
-- rest points into text and keeps nothing alive, so text must stay reachable.
print(value:asNumber(), rest:asString()) -- 1234 abc
```

### Strings

```luau
local buf = c.new(c.char:array(16))
buf:write("hello") -- the bytes are copied, and the rest of buf stays zero
print(buf:asString()) -- hello

-- char *getenv(const char *name);
local getenv = ffi.process:load("getenv", c.func(c.char:ptr(), c.char:const():ptr()))
local value = getenv("NO_SUCH_VARIABLE")
print(value:isNull()) -- true, and value == nil would be false
```

### What keeps memory alive

```luau
local arr = c.new(c.int:array(3), { 10, 20, 30 })
local second = arr[1] -- a view keeps the array's memory
local ref = c.new(c.int, 7):ref() -- a pointer from :ref() keeps the int's memory
arr = nil
print(second:asNumber(), ref:deref():asNumber()) -- 20 7

local fileT = c.opaque("FILE")
-- FILE *tmpfile(void);
local tmpfile = ffi.process:load("tmpfile", c.func(fileT:ptr()))
-- int fclose(FILE *f);
local fclose = ffi.process:load("fclose", c.func(c.int, fileT:ptr()))
-- int setvbuf(FILE *f, char *buf, int mode, size_t size);
local setvbuf = ffi.process:load("setvbuf", c.func(c.int, fileT:ptr(), c.char:ptr(), c.int, c.size_t))

-- C uses buffer until fclose. Attached, it is freed only after the finalizer has returned.
local file = c.gc(tmpfile(), fclose)
local buffer = c.new(c.char:array(4096))
setvbuf(file, buffer, 0, c.sizeof(buffer)) -- mode 0 is _IOFBF
c.attach(buffer, file)
```

### Conversions and equality

```luau
print((pcall(c.new, c.int, 1.5))) -- false, an implicit conversion raises
print(c.cast(300, c.uchar):asNumber()) -- 44, only c.cast wraps

local a, b = c.new(c.int, 1), c.new(c.int, 1)
print(a == b, rawequal(a, b), c.equals(a, b)) -- true false true
```

## Drawbacks and safety

Native code remains trusted. The checks above catch mistakes in declarations and values. A wrong signature, an invalid address, a use after free or unbounded pointer access can still corrupt memory. Enabling FFI is not a sandbox boundary.

`LUTE_ENABLE_FFI` is on by default, which builds libffi into Lute. Turning it off excludes libffi and the definitions and makes the module fail to open. Permission policy remains a separate decision for Lute.

Views create userdata, so fine-grained access costs more than bulk byte access.

## Alternatives

A smaller interface of numbers, pointers and buffers, as in Deno and Bun, would leave struct layout and every lifetime to the script. Parsing C declarations would need a parser and would bypass Luau's type functions. Compiled plugins remain an alternative for bindings that need native code or callbacks. A Luau finalizer can already hold what C keeps in its upvalues, and the runtime holds them until it has run. `c.attach` holds only native memory, so it can refuse cycles and order finalizers when a VM closes, which upvalues cannot.

## Out of scope

Callbacks, a nonblocking call mode, varargs, custom calling conventions, packed and explicitly aligned structs, pointer bounds, arithmetic operators, and bulk memory helpers are deferred. So are freeing runtime memory early, handing it to C, a size hint for `c.gc`, and finalizers on views, tied to the enclosing allocation. Header parsing, bitfields, `long double`, and complex types are outside this proposal.

Two later milestones can add lifetime guarantees without changing this interface. Stored-pointer tracking would let a pointer stored in runtime memory keep its target alive, as if each store attached the target to the memory it was stored in. Stores can form cycles, which `c.attach` refuses, so tracking needs collector support for cycles. An address registry would let a pointer from C into runtime memory find that memory, keep it alive and get bounds. Neither sees a pointer C keeps in its own memory, which stays the case for `c.attach`.

## Implementation milestones

Each milestone is a separate PR once this proposal is agreed. It builds on the earlier ones and passes its own tests when it lands. Errno capture and direct string arguments belong to Calls because they are part of performing a call. Lifetime follows Calls because `c.gc` can take a C function as its finalizer.

| Milestone | Review scope |
| --- | --- |
| 1. Types | The C type system: primitive types, the `ptr`, `array` and `const` methods, structs, functions, opaque types, native layout, structural equality |
| 2. Values | `CData`, construction, writes, primitive and byte conversions with range checks, pointer write checks, const storage, equality, `CAddress`, collector pacing |
| 3. Access | Views, fields, indexing, dereference, address-of, casts, unions, allocation shorthand |
| 4. Loader | Library handles, deferred unloading, process symbols, pointer size |
| 5. Calls | `lib:load`, libffi calls, return conversion, Luau strings for `const char*` parameters, errno and last-error capture, argument types from the signature |
| 6. Lifetime | `c.gc` finalizers through the scheduler, `c.string`, `c.attach` and `c.detach` |

Each milestone has tests, with layout and calls checked against a compiled C fixture. Documentation examples must run as well as typecheck.
