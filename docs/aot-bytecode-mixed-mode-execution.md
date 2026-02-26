# Mixing AOT-Compiled Dart Code with a Bytecode Interpreter

## Executive Summary

The Dart VM already contains the infrastructure to run **AOT-compiled native code
alongside a bytecode interpreter** in the same isolate. This capability is gated
behind the `DART_DYNAMIC_MODULES` compile-time flag and uses a pair of stubs
(`InterpretCall` and `InvokeDartCodeFromBytecode`) to shuttle control between the
two worlds. This document explains the complete mechanism, identifies every key
integration point, and describes what would be required to use this for an
"extern function" model where native AOT code calls into methods only available
as bytecode.

---

## Table of Contents

1. [Existing Infrastructure Overview](#1-existing-infrastructure-overview)
2. [Bootstrapping a Mixed-Mode Runtime](#2-bootstrapping-a-mixed-mode-runtime)
3. [The InterpretCall Stub: AOT to Interpreter](#3-the-interpretcall-stub-aot-to-interpreter)
4. [The InvokeDartCodeFromBytecode Stub: Interpreter to AOT](#4-the-invokedartcodefrombytecode-stub-interpreter-to-aot)
5. [The Interpreter Dispatch Loop](#5-the-interpreter-dispatch-loop)
6. [The AOT Dispatch Table and Virtual Calls](#6-the-aot-dispatch-table-and-virtual-calls)
7. [Implementing "Extern" Functions (Native Calling Bytecode)](#7-implementing-extern-functions-native-calling-bytecode)
8. [Loading Bytecode Into a Running AOT Isolate](#8-loading-bytecode-into-a-running-aot-isolate)
9. [Stack Frame Interleaving and Unwinding](#9-stack-frame-interleaving-and-unwinding)
10. [Object Pool and Constant Management](#10-object-pool-and-constant-management)
11. [Garbage Collection Considerations](#11-garbage-collection-considerations)
12. [SwitchableCall Resolution Chain in AOT (Deep Dive)](#12-switchablecall-resolution-chain-in-aot-deep-dive)
13. [Interpreter-Side Instance Call Resolution](#13-interpreter-side-instance-call-resolution)
14. [Practical Concerns and Tradeoffs](#14-practical-concerns-and-tradeoffs)
15. [Summary of Key Source Locations](#15-summary-of-key-source-locations)

---

## 1. Existing Infrastructure Overview

The Dart VM supports two execution backends that can coexist within the same
process:

- **AOT-compiled native code** -- functions pre-compiled to machine instructions
  during `gen_snapshot`, stored as bare instruction payloads in the snapshot
  image, dispatched via the per-isolate-group Dispatch Table or direct/static
  call sites.

- **Kernel Bytecode (KBC) interpreter** -- a register-based bytecode interpreter
  (`runtime/vm/interpreter.cc`) that executes `Bytecode` objects loaded at
  runtime. Uses computed-goto dispatch on supported platforms.

The bridge is controlled by a single compile-time flag:

```
DART_DYNAMIC_MODULES   (defined via dart_dynamic_modules = true in runtime_args.gni)
```

When this flag is **off**, `Function::IsInterpreted()` is hardcoded to `false`,
and none of the interpreter, bytecode reader, or bridging stubs are compiled in.
When it is **on**, the VM gains the ability to:

1. Load bytecode modules at runtime (`BytecodeLoader`).
2. Attach bytecode to `Function` objects.
3. Redirect compiled call sites through the interpreter transparently.

**Key insight**: A function's `code_` field serves as the mode selector. It can
point to:
- `StubCode::LazyCompile()` -- not yet compiled (JIT only)
- A real `Code` object -- AOT- or JIT-compiled native instructions
- `StubCode::InterpretCall()` -- interpreted via the bytecode engine

The check is trivial (pointer comparison), making the dispatch at call sites
essentially free.

### Relevant source files

| Area | Path |
|---|---|
| Interpreter engine | `runtime/vm/interpreter.{h,cc}` |
| Bytecode constants | `runtime/vm/constants_kbc.{h,cc}` |
| Bytecode reader/loader | `runtime/vm/bytecode_reader.{h,cc}` |
| Bytecode frame layout | `runtime/vm/stack_frame_kbc.h` |
| InterpretCall C function | `runtime/vm/runtime_entry.cc:4816` |
| InterpretCall stub (x64) | `runtime/vm/compiler/stub_code_compiler_x64.cc:3182` |
| InvokeDartCodeFromBytecode (x64) | `runtime/vm/compiler/stub_code_compiler_x64.cc:1748` |
| Function::AttachBytecode | `runtime/vm/object.cc:8555` |
| Function::IsInterpreted | `runtime/vm/object.cc:8575` |
| Dispatch Table | `runtime/vm/dispatch_table.h` |
| Dispatch Table Generator | `runtime/vm/compiler/aot/dispatch_table_generator.{h,cc}` |
| Dynamic module loading | `runtime/lib/object.cc:557` (Internal_loadDynamicModule) |
| Build flag | `runtime/runtime_args.gni:79` |

---

## 2. Bootstrapping a Mixed-Mode Runtime

### 2.1 Build configuration

The runtime must be built with `dart_dynamic_modules = true` in
`runtime/runtime_args.gni`. This causes `DART_DYNAMIC_MODULES` to be defined,
which includes the interpreter, the bytecode reader, and both bridging stubs in
the resulting binary.

Critically, this flag is **orthogonal** to `DART_PRECOMPILED_RUNTIME`. A binary
can be both an AOT runtime *and* have the interpreter compiled in. This is
exactly the mixed-mode scenario.

### 2.2 VM initialization path

The normal AOT boot sequence (`Dart::Init` in `runtime/vm/dart.cc`) remains
unchanged:

1. OS threads, zones, timeline, isolate infrastructure initialized.
2. VM isolate created; AOT snapshot deserialized (`kFullAOT` kind).
3. `StubCode` entries are deserialized from the snapshot (not generated).
4. `Object::FinishInit` and `DispatchTable` loaded.
5. `InstructionsTable` built from the code section of the image.

For mixed-mode, two additions are needed:

- **Include the InterpretCall and InvokeDartCodeFromBytecode stubs** in the AOT
  snapshot. These are already in `stub_code_list.h` (lines 84 and 93) and will
  be serialized when `DART_DYNAMIC_MODULES` is enabled. The precompiler must
  ensure they are treated as roots.

- **Initialize the interpreter per-thread**. Each mutator thread in an isolate
  that will run interpreted code needs an `Interpreter` instance. The interpreter
  owns its own stack (a malloc'd region growing upward), a pool pointer
  (`pp_`), a lookup cache, and setjmp buffers for exception handling. The
  `Interpreter::Current()` accessor reads it from Thread-local storage.

### 2.3 Interpreter allocation

The interpreter is not created eagerly for every thread. In the existing
`DART_DYNAMIC_MODULES` code, it is created on demand when bytecode is first
invoked. The `Interpreter::Current()` call lazily allocates.

### 2.4 The `interpret_call_entry_point_` thread field

The `Thread` object caches the C function pointer of `InterpretCall` at offset
`Thread::interpret_call_entry_point_offset()` (defined in `thread.h:266`). This
avoids an indirect lookup when the stub needs to call the interpreter.

---

## 3. The InterpretCall Stub: AOT to Interpreter

This is the **primary mechanism for calling an interpreted function from
AOT-compiled code**. It is the equivalent of a calling convention adapter.

### 3.1 How it gets wired up

When bytecode is attached to a function:

```
Function::AttachBytecode(bytecode):
  1. Store bytecode in ic_data_array_or_bytecode_ field.
  2. Call SetInstructions(StubCode::InterpretCall()).
     → This sets code_ to the InterpretCall Code object.
     → This also sets entry_point_ to InterpretCall's entry.
```

(`runtime/vm/object.cc:8555-8567`)

From this point on, any compiled code that calls this function -- whether through
a static call, a dispatch table call, or a switchable call -- will land in the
InterpretCall stub, because all those mechanisms ultimately jump to
`function.entry_point()`.

### 3.2 The stub's operation (x64 example)

`runtime/vm/compiler/stub_code_compiler_x64.cc:3182-3268`:

```
InterpretCall stub:
  1. EnterStubFrame
  2. Compute argc from arguments descriptor (accounting for type args)
  3. Compute argv pointer (address of first argument on stack)
  4. Negate argc (convention: negative argc = descending memory order)
  5. Align stack for C calling convention
  6. Move arguments into C calling convention registers:
       arg1 = FUNCTION_REG (the Function object)
       arg2 = ARGS_DESC_REG (ArgumentsDescriptor)
       arg3 = argc (negative)
       arg4 = argv (pointer into Dart stack)
       arg5 = THR (Thread*)
  7. Save top_exit_frame_info (so stack walker can find this frame)
  8. Mark exit_through_ffi = runtime_call
  9. Load interpret_call_entry_point_ from Thread
  10. call RAX  (calls the C function InterpretCall)
  11. Restore VMTag to kDartTagId
  12. Reset exit_through_ffi and top_exit_frame_info
  13. LeaveStubFrame
  14. ret (result in RAX)
```

### 3.3 The C function `InterpretCall`

`runtime/vm/runtime_entry.cc:4816-4856`:

This is a plain C function (not a runtime entry in the traditional sense). It:

1. Obtains `Interpreter::Current()`.
2. Calls `interpreter->Call(function, argdesc, argc, argv, ...)`.
3. Checks the result for errors; if error, transitions to VM state and
   propagates.
4. Returns the result as a `uword` directly to the stub.

**Important**: The thread stays in `kThreadInGenerated` state throughout. The
interpreter is considered "generated code" from the VM's perspective. This is
deliberate -- it avoids safepoint transitions on every interpreted call.

### 3.4 What makes this work for AOT dispatch

Because `Function::entry_point_` is set to the InterpretCall stub entry, **all
existing dispatch mechanisms work unchanged**:

- **Static calls** jump directly to `entry_point_`.
- **Dispatch table calls** store `entry_point_` in the table slot. The generated
  `call [table + cid*8 + offset]` lands in the stub.
- **Switchable calls** use the `entry_point_` cached in the Function/Code, so
  monomorphic, polymorphic, and megamorphic dispatch all correctly route to the
  stub.

No special casing is required at call sites.

---

## 4. The InvokeDartCodeFromBytecode Stub: Interpreter to AOT

When the interpreter needs to call a function that has compiled native code, it
uses the `InvokeDartCodeFromBytecode` stub. This is the reverse transition.

### 4.1 The interpreter's `Invoke` decision point

`runtime/vm/interpreter.cc:725-754`:

```
Interpreter::Invoke(thread, call_base, call_top, pc, FP, SP):
  loop:
    if Function::IsInterpreted(function):
      → InvokeBytecode(...)  // stay in interpreter
    else if Function::HasCode(function):
      → InvokeCompiled(...)  // exit to compiled code
    else:
      → CompileFunction runtime call, then retry
```

This three-way dispatch is the heart of mixed-mode execution.

### 4.2 InvokeCompiled: the interpreter side

`runtime/vm/interpreter.cc:594-690`:

1. Call `Exit(thread, *FP, call_top+1, *pc)` -- this saves the interpreter's
   frame pointer and PC into the exit frame, and sets
   `thread->top_exit_frame_info` so the stack walker can bridge the gap.

2. Set up a `InterpreterSetjmpBuffer` for exception handling.

3. Call the `InvokeDartCodeFromBytecode` stub via a C function pointer:

   ```
   result = entrypoint(
     function->entry_point_,    // in AOT mode
     argdesc_,
     call_base,                 // pointer to first argument
     thread
   );
   ```

4. On return, call `Unexit(thread)` to restore the interpreter's PC and FP.

5. Restore the object pool: `pp_ = FrameBytecode(*FP)->object_pool()`.

6. Push the result onto the interpreter stack and continue dispatching.

### 4.3 The InvokeDartCodeFromBytecode stub (x64)

`runtime/vm/compiler/stub_code_compiler_x64.cc:1748-1894`:

This is a full entry stub similar to `InvokeDartCode`, but specifically designed
to be called from the interpreter (C calling convention on entry, not from
generated Dart code):

```
InvokeDartCodeFromBytecodeStub:
  1. EnterFrame (native frame, not stub frame)
  2. Push code object to PC marker slot
  3. Push arguments descriptor (later replaced by Smi argc)
  4. Push callee-saved registers
  5. Set up THR from argument register
  6. Save VMTag, top_resource, exit_through_ffi, top_exit_frame_info
  7. Set VMTag = kDartTagId (we're entering compiled Dart code)
  8. Load arguments descriptor into ARGS_DESC_REG
  9. Push arguments from call_base onto the native stack
  10. In precompiled mode: load global_object_pool into PP
  11. call target_entry_point  (the compiled function)
  12. Pop arguments, restore exit frame info
  13. Restore VMTag, resources
  14. Pop callee-saved registers
  15. LeaveFrame, ret
```

The key difference from `InvokeDartCode` is that this stub:
- Is called from C code (interpreter), not from generated Dart code.
- Expects C calling convention arguments.
- Sets up the entry frame markers that `StackFrameIterator` understands.

---

## 5. The Interpreter Dispatch Loop

The KBC interpreter (`runtime/vm/interpreter.cc`) is a ~4500-line dispatch loop
using computed-goto for performance.

### 5.1 Stack model

The interpreter maintains its **own stack**, separate from the native C/machine
stack. This stack grows **upward** (from low to high addresses), which enables
efficient unsigned indexing for local variables.

Frame layout per `stack_frame_kbc.h`:

```
  FP[-4]  function
  FP[-3]  bytecode (code marker)
  FP[-2]  saved caller PC (bytecode instruction pointer)
  FP[-1]  saved caller FP
  FP[0]   first local / suspend state
  FP[1..] more locals
  ...
  SP      top of stack (grows up)
```

### 5.2 Call instructions in bytecode

The interpreter supports several call variants:

| Bytecode | Description |
|---|---|
| `DirectCall D, F` | Known target; loads function and argdesc from pool[D], argc=F |
| `InterfaceCall D, F` | Virtual dispatch; resolves via lookup cache or runtime |
| `DynamicCall D, F` | Dynamic dispatch by name |
| `UncheckedClosureCall D, F` | Calls closure's function directly |
| `ExternalCall D` | Calls a native/FFI function |

All Dart-level calls go through `Interpreter::Invoke()`, which performs the
IsInterpreted/HasCode check described above.

### 5.3 Lookup cache

For interface/dynamic calls, the interpreter maintains a 1024-entry hash table
(`LookupCache`) keyed on `(receiver_cid, target_name, argdesc)` → `target
Function`. On miss, it falls through to `DRT_InterpretedInstanceCallMissHandler`
runtime entry.

---

## 6. The AOT Dispatch Table and Virtual Calls

### 6.1 How the dispatch table works

In AOT mode, virtual method calls use a **dispatch table** -- a flat array of
`uword` entry points indexed by `[selector_offset + class_id]`. Each Thread
caches a pointer to this table at `Thread::dispatch_table_array_offset()`.

Generated code for a virtual call:

```
  LoadClassId(cid_reg, receiver)
  LoadDispatchTable(table_reg)    // table_reg = thread->dispatch_table_array_
  call [table_reg + cid*8 + (selector_offset - kOriginElement)*8]
```

The table is built during AOT compilation by `DispatchTableGenerator`
(`runtime/vm/compiler/aot/dispatch_table_generator.cc`). For each
`(selector_id, class_id)` pair, it stores the entry point of the implementing
function's Code object.

### 6.2 Making the dispatch table work with interpreted functions

**This works automatically.** When bytecode is attached to a function via
`AttachBytecode`, its `entry_point_` is set to the `InterpretCall` stub. The
dispatch table stores `entry_point_` values. So for an interpreted
implementation of a virtual method, the table slot will contain the InterpretCall
stub's address.

When compiled code performs a dispatch table call and lands on an interpreted
function, the InterpretCall stub fires, enters the interpreter, executes the
bytecode, and returns the result. The compiled caller is completely unaware that
the callee was interpreted.

**However**, the dispatch table is built statically during AOT compilation. For
functions that are *not known at compile time* (i.e., provided only as bytecode
later), the dispatch table has no entries. This is where the "extern function"
problem arises (see Section 7).

### 6.3 Extending the dispatch table at runtime

To support dynamically loaded bytecode that implements new virtual method
overrides:

1. **Allocate new class IDs** for classes introduced by bytecode. The
   `ClassTable` can grow dynamically (`runtime/vm/class_table.h`).

2. **Extend the dispatch table**. The table needs to be reallocated to
   accommodate new CIDs. For each selector that the new class implements, the
   entry at `[selector_offset + new_cid]` must be set to the InterpretCall
   stub's entry point.

3. **Atomic replacement**. The old table pointer in `Thread` must be atomically
   swapped to the new one to avoid races with mutator threads.

This is the approach that the existing `DART_DYNAMIC_MODULES` infrastructure
would use. The dispatch table generator produces the initial table; runtime
extension adds new rows.

---

## 7. Implementing "Extern" Functions (Native Calling Bytecode)

This section addresses the core question: how to have AOT-compiled code call
functions that are **not known at compile time** and will only be provided as
bytecode at runtime.

### 7.1 The problem statement

At AOT compile time, we want to declare that certain functions are "extern" --
they have a known signature and name, but no implementation. The implementation
will be supplied later as bytecode. Compiled code should be able to call these
functions directly.

### 7.2 Approach A: Placeholder Function with LazyCompile-style stub

**Concept**: At AOT compile time, create a `Function` object for each extern
declaration. Set its `code_` to a custom stub (like `InterpretCall` or a new
`ExternBytecodeCall` stub). At runtime, when bytecode is loaded, attach the
bytecode to these placeholder functions.

**How it works**:

1. **At compile time**: The precompiler creates Function objects for extern
   declarations. Their code is set to a **sentinel stub** that will either:
   - Trap (if bytecode hasn't been loaded yet), or
   - Redirect to InterpretCall (if bytecode is attached).

   The simplest approach: set `code_` to `StubCode::InterpretCall()` from the
   start, and leave `ic_data_array_or_bytecode_` as null. The InterpretCall C
   function checks `Function::HasBytecode(function)` in debug mode; we'd need a
   runtime check that throws a "not yet loaded" error if bytecode is null.

2. **At link time / dispatch table**: For virtual methods, the dispatch table
   slot for this function's `(selector, cid)` pair is filled with the
   InterpretCall stub entry. For static calls, the call site directly references
   the Function's entry point, which is the InterpretCall stub.

3. **At runtime**: `BytecodeLoader::LoadBytecode()` reads the bytecode module,
   finds the matching Function objects (by name or a predefined mapping), and
   calls `Function::AttachBytecode()`. This stores the Bytecode object in
   `ic_data_array_or_bytecode_` but *does not change* `entry_point_` (it already
   points to InterpretCall).

4. **On call**: Compiled code jumps to `entry_point_` → InterpretCall stub →
   `InterpretCall` C function → `Interpreter::Call()` → bytecode execution →
   result returned to compiled caller.

**Advantages**: Minimal changes to the existing compilation pipeline. The
Function object exists at compile time, so all call sites can be resolved
normally.

**Disadvantages**: Requires that extern functions are declared in the Dart
source at compile time (even if not implemented). The precompiler must be taught
not to optimize them away.

### 7.3 Approach B: Runtime function registration

**Concept**: At AOT compile time, call sites reference a **function slot** (an
index into an indirection table). At runtime, bytecode loading fills these slots
with actual function entry points.

**How it works**:

1. **At compile time**: For each extern function, allocate an entry in a
   "extern function table" (similar to a PLT/GOT in traditional linkers). Call
   sites emit an indirect call through this table.

2. **At runtime**: When bytecode is loaded, the table entries are filled with
   the InterpretCall stub entry. The Function objects are created and have
   bytecode attached.

3. **On call**: The indirect call goes through the table → InterpretCall stub →
   interpreter.

**This mirrors how `SwitchableCall` sites already work**. An unlinked call site
starts with `UnlinkedCall` data and gets patched on first call. We could use the
same `SwitchableCallMiss` mechanism to lazily resolve extern functions to their
bytecode implementations.

### 7.4 Approach C: Leveraging the existing SwitchableCall mechanism

The most elegant approach for extern *instance* methods uses the existing
switchable call infrastructure:

1. **At compile time**: Emit normal switchable call sites. For classes that will
   be provided by bytecode, the dispatch table entries are initialized to the
   null_error_stub or a custom "not-yet-loaded" stub.

2. **At runtime**: When bytecode is loaded:
   - Register new classes in the ClassTable.
   - Extend the dispatch table with entries pointing to InterpretCall for the
     new class methods.
   - For switchable call sites that have already cached a specific target, they
     will naturally fall through to `SwitchableCallMiss` when they encounter a
     receiver of the new class, and the miss handler will resolve to the
     interpreted function.

3. **For static calls**: Use Approach A (placeholder Function objects).

### 7.5 Recommended approach

**Use Approach A for static/direct calls, Approach C for virtual calls.** This
minimizes changes:

- Extern functions are declared in Dart with `external` keyword or a custom
  annotation.
- The precompiler creates Function objects for them with InterpretCall as their
  code.
- Virtual dispatch works through the dispatch table (extended at runtime).
- Static dispatch works through the Function's entry point (InterpretCall from
  the start).
- `BytecodeLoader` attaches bytecode to the pre-existing Function objects.

The key modification is teaching the precompiler to:
1. Not tree-shake extern function declarations.
2. Emit InterpretCall as their code in the AOT snapshot.
3. Include dispatch table slots for extern virtual methods.

---

## 8. Loading Bytecode Into a Running AOT Isolate

### 8.1 The existing loading path

The `loadDynamicModule` API (`sdk/lib/_internal/vm/lib/internal_patch.dart`)
provides the user-facing entry:

```dart
Future<Object?> loadDynamicModule({Uri? uri, Uint8List? bytes})
```

The native implementation (`runtime/lib/object.cc:557`) calls:

```
BytecodeLoader(thread, typed_data).LoadBytecode()
```

### 8.2 BytecodeLoader operation

`BytecodeLoader` (`runtime/vm/bytecode_reader.{h,cc}`) performs:

1. Parse the bytecode binary (a structured format with string tables, object
   tables, library/class/member declarations, and bytecode payloads).

2. For each library in the module:
   - Create or find the Library object.
   - Register classes in the ClassTable (assigns new CIDs if needed).
   - Read field and function declarations.

3. For each function:
   - Create a Bytecode object containing the instruction payload and object pool.
   - Attach bytecode via `Function::AttachBytecode()`.
   - This sets `entry_point_` to InterpretCall.

4. Return the entry point function for invocation.

### 8.3 What must change for extern function resolution

When loading bytecode for an extern function, the loader must:

1. **Find the pre-existing Function object** (created at AOT compile time)
   rather than creating a new one. This can be done by looking up the function
   by its library URI, class name, and function name in the existing
   ObjectStore/ClassTable.

2. **Call AttachBytecode** on the existing Function. Since its `entry_point_`
   already points to InterpretCall, no call site patching is needed.

3. **For new classes** (not just extern functions on existing classes):
   - Register in ClassTable.
   - Extend the dispatch table.
   - Create the necessary type metadata.

### 8.4 Thread safety

The program lock (`IsolateGroup::program_lock()`) must be held as a writer
during bytecode attachment. This is already required by `AttachBytecode`:

```cpp
ASSERT(IsolateGroup::Current()->program_lock()->IsCurrentThreadWriter());
```

Dispatch table extension must also be atomic with respect to mutator threads.
The existing mechanism uses `SafepointWriteRwLocker`.

---

## 9. Stack Frame Interleaving and Unwinding

In a mixed-mode call chain, the native and interpreter stacks interleave:

```
[Compiled Frame A]
[Compiled Frame B]
   ↓ calls interpreted function
[InterpretCall StubFrame]
   ↓ enters interpreter
[Interpreter Entry Frame]  (on interpreter's own stack)
[Bytecode Frame C]
[Bytecode Frame D]
   ↓ calls compiled function
[Exit Frame]              (marks interpreter exit)
[InvokeDartCodeFromBytecode Entry Frame]  (on native stack)
[Compiled Frame E]
   ↓ returns to interpreter
[Bytecode Frame D continues]
```

### 9.1 How the stack walker handles this

The `StackFrameIterator` (`runtime/vm/stack_frame.h`) uses
`top_exit_frame_info` to bridge between compiled and interpreter frames. When
the interpreter calls compiled code via `InvokeCompiled`, it calls `Exit()`
which sets `thread->top_exit_frame_info` to the interpreter's exit frame. The
stack walker follows this chain:

1. Walk compiled frames normally.
2. When encountering an `InvokeDartCodeFromBytecode` entry frame, switch to
   interpreter frame walking using the saved exit frame info.
3. When encountering an `InterpretCall` stub frame, return to compiled frame
   walking.

### 9.2 Exception unwinding

Exceptions propagate correctly across the boundary:

- **Exception in interpreted code**: The interpreter's `HandleException` block
  uses `InterpreterSetjmpBuffer` (linked list of setjmp points) to unwind to the
  correct handler. If no handler is found within interpreted frames, the
  exception propagates to the compiled caller via `InterpretCall`'s error check.

- **Exception in compiled code called from interpreter**: The compiled
  exception handler runs as normal. If unhandled, `InvokeCompiled` detects the
  error result and unwinds the interpreter stack.

---

## 10. Object Pool and Constant Management

### 10.1 Two pool systems

- **Compiled code**: In AOT mode, uses a **global object pool** shared across
  all compiled functions. Stored at `Thread::global_object_pool_offset()` and
  loaded into the `PP` register.

- **Interpreter**: Each `Bytecode` object has its **own object pool**. The
  interpreter maintains `pp_` as a C++ member, switching it on every function
  entry:

  ```
  pp_ = bytecode->untag()->object_pool();
  ```

### 10.2 Pool switching at boundaries

- **Compiled → Interpreted**: The InterpretCall stub does not touch PP. The
  interpreter loads the callee's pool in `Interpreter::Call()`.

- **Interpreted → Compiled**: The `InvokeDartCodeFromBytecode` stub loads the
  global object pool into PP before calling the target function. On return, the
  interpreter restores its own pool:

  ```
  pp_ = InterpreterHelpers::FrameBytecode(*FP)->untag()->object_pool();
  ```

### 10.3 Implications for extern functions

Bytecode object pools are populated by the `BytecodeLoader` during loading. They
contain references to classes, functions, types, and literal constants. For
extern functions, the bytecode's object pool must reference the AOT-compiled
program's classes and types correctly. This requires the loader to resolve
references against the existing isolate's ClassTable and ObjectStore rather than
creating new objects.

---

## 11. Garbage Collection Considerations

### 11.1 Root scanning

The GC must find all live objects on both stacks:

- **Compiled frames**: Scanned using `CompressedStackMaps` that map PC ranges
  to live pointer locations.

- **Interpreter frames**: The `Interpreter::VisitObjectPointers()` method scans
  `pp_` (pool pointer), `argdesc_` (arguments descriptor), and
  `subtype_test_cache_`. The interpreter stack itself is scanned by the GC
  through exit frame markers.

- **Bytecode objects**: Treated as normal heap objects. The GC follows
  `Bytecode → object_pool → ...` chains normally.

### 11.2 Safepoints

The interpreter runs in `kThreadInGenerated` state, meaning it does *not*
automatically enter safepoints between instructions. Stack overflow checks
(`CheckStack` bytecode) serve as safepoint poll points. The interpreter's
`CheckStack` handler checks `thread->stack_limit()` which may be set to trigger
a safepoint.

---

## 12. SwitchableCall Resolution Chain in AOT (Deep Dive)

This section provides a detailed walk-through of exactly what happens when an
AOT call site encounters a receiver for which the target is an interpreted
function. Understanding this chain is critical for the "extern function" model.

### 12.1 AOT call site lifecycle

Every instance call site in AOT code starts life as an `UnlinkedCall`. The call
site data is a pair `(data, entry_point)` stored inline at the call site:

```
Initial state:
  data  = UnlinkedCall { target_name, arguments_descriptor }
  entry = StubCode::SwitchableCallMiss().MonomorphicEntryPoint()
```

On the **first call**, the `SwitchableCallMiss` stub fires and enters
`DRT_SwitchableCallMiss` (`runtime/vm/runtime_entry.cc:3415`). The handler:

1. Walks the stack to find the caller frame and its Code.
2. Reads the call site data via `CodePatcher::GetSwitchableCallDataAt`.
3. Creates a `PatchableCallHandler` and calls `ResolveSwitchAndReturn(old_data)`.
4. `ResolveTargetFunction` extracts the method name and arguments descriptor
   from the `UnlinkedCall`, then calls `Resolver::ResolveDynamicForReceiverClass`
   to find the actual target function for the receiver's class.
5. `HandleMissAOT` dispatches based on the old data's class ID. For
   `kUnlinkedCallCid` it calls `DoUnlinkedCallAOT`.

### 12.2 `DoUnlinkedCallAOT` and patching

`runtime/vm/runtime_entry.cc:2763-2805`:

`DoUnlinkedCallAOT` creates an ICData and transitions the call site. If the
target function has been resolved (not null) and its prologue doesn't need an
arguments descriptor, the call site is patched to monomorphic:

```
DoUnlinkedCallAOT(unlinked, target_function):
  1. Create ICData with the receiver CID → target_function mapping.
  2. If target doesn't need ARGS_DESC_REG:
     → Patch to monomorphic: data = Smi(receiver_cid), code = target Code
     → Or MonomorphicSmiableCall if receiver can be Smi.
  3. If target needs ARGS_DESC_REG:
     → Patch to ICCallThroughCode (carries the ICData).
  4. Return ICData via ReturnAOT for the miss stub to continue.
```

**Key insight for interpreted functions**: When `target_function` has its code
set to `StubCode::InterpretCall()`, `target_function.CurrentCode()` returns the
InterpretCall Code object. The call site gets patched to point to the
InterpretCall Code's entry point. Subsequent calls for the same receiver class
go directly to InterpretCall without any miss handler overhead.

### 12.3 Progression through dispatch states

Call sites progress through increasingly general dispatch states:

```
UnlinkedCall → Monomorphic → SingleTargetCache → ICData → MegamorphicCache
```

- **Monomorphic** (`kSmiCid`): Checks `receiver_cid == expected_cid`, jumps to
  cached entry point (which may be InterpretCall's entry).
- **SingleTargetCache**: Checks `lower_cid <= receiver_cid <= upper_cid`, used
  when a range of CIDs share the same target function.
- **ICData** (via `ICCallThroughCode`): Array of `(cid, target)` pairs.
- **MegamorphicCache**: Hash table lookup for large polymorphic sites.

At **every state**, if the target function is interpreted, its entry point is
the InterpretCall stub, and this is what gets stored/cached. No special handling
for interpreted functions is needed at any stage.

### 12.4 Implications for dynamically loaded classes

When bytecode loads a new class with a new CID, existing monomorphic/single-
target call sites for the same selector will miss (CID doesn't match). The miss
handler will:

1. Resolve the method for the new class (finding the interpreted function).
2. Widen the call site (monomorphic → single-target or ICData).
3. Cache the InterpretCall entry for the new CID.

This happens automatically through the existing switchable call mechanism.
No special code is needed.

---

## 13. Interpreter-Side Instance Call Resolution

### 13.1 The `InterpretedInstanceCallMissHandler`

When the interpreter's lookup cache misses during an `InterfaceCall` or
`DynamicCall` bytecode, it calls `DRT_InterpretedInstanceCallMissHandler`
(`runtime/vm/runtime_entry.cc:3456-3486`):

```
InterpretedInstanceCallMissHandler(receiver, target_name, arg_desc):
  1. Finalize receiver's class if needed.
  2. Call Resolver::ResolveDynamicForReceiverClass.
  3. If not found, fall back to InlineCacheMissHelper (noSuchMethod dispatch).
  4. Return the resolved Function.
```

The interpreter then updates its lookup cache with `(receiver_cid, target_name,
arg_desc) → Function`. On subsequent calls with the same receiver CID, the
cache hit path directly calls `Interpreter::Invoke()`.

### 13.2 The `Invoke` decision for resolved targets

After resolving, `Interpreter::Invoke()` checks the target:

- **Interpreted target** (`IsInterpreted(function)` is true): Calls
  `InvokeBytecode` -- stays entirely within the interpreter, using bytecode
  frame linkage.
- **Compiled target** (`HasCode(function)` is true): Calls `InvokeCompiled` --
  exits to native code via the `InvokeDartCodeFromBytecode` stub.
- **Neither**: Calls `DRT_CompileFunction` to compile or load bytecode, then
  retries.

This means the interpreter can seamlessly call both AOT functions and other
interpreted functions.

### 13.3 `ExternalCall` bytecode: interpreter calling native C functions

The `ExternalCall D` bytecode instruction (`runtime/vm/interpreter.cc:2388-2436`)
handles calls to native/FFI functions from within interpreted code:

```
ExternalCall D:
  1. Load trampoline and native_function from constant pool[D] and pool[D+1].
  2. If null (not yet resolved):
     → Call DRT_ResolveExternalCall runtime entry.
     → Reload from pool after resolution.
  3. Set up NativeArguments.
  4. Call InvokeNative(thread, interpreter, trampoline, native_function, args).
  5. Handle result/exception.
```

This is the interpreter's equivalent of FFI calls in compiled code. The
trampoline handles the calling convention translation.

---

## 14. Practical Concerns and Tradeoffs

### 14.1 Performance

Interpreted code is significantly slower than AOT-compiled code (roughly 10-50x
for computation-heavy code). The transition cost between modes is moderate:

- **AOT → Interpreted**: One stub frame setup + one C function call +
  interpreter setup. ~100-200ns overhead per call.
- **Interpreted → AOT**: One `Exit` + one stub + one compiled function call.
  Similar overhead.

For hot paths, the interpreter's lookup cache (1024 entries) helps with virtual
call resolution, but it is far less effective than AOT's dispatch table or
inline caches.

### 14.2 Memory overhead

The interpreter adds:
- Per-thread interpreter stack (configurable size).
- Bytecode objects with their own object pools.
- The bytecode reader infrastructure.

Bytecode is typically more compact than native code (~20-50% of native code
size), which may offset the interpreter overhead for large programs.

### 14.3 Debugging and profiling

The interpreter supports:
- **Single-stepping**: Via a single-step flag check in every dispatch.
- **Breakpoints**: Bytecode instructions can be replaced with breakpoint opcodes
  (`VMInternal_Breakpoint_*`), sized to match the original instruction.
- **Profiling**: VMTag switching (kDartInterpretedTagId vs kDartTagId) allows
  profilers to distinguish interpreted from compiled code.
- **Stack traces**: The stack walker correctly produces combined traces spanning
  both compiled and interpreted frames.

### 14.4 Async/await support

The interpreter supports `Suspend` and `EntrySuspendable` bytecodes. Interpreted
async functions create `SuspendState` objects just like compiled ones. The
`Interpreter::Resume()` method handles resumption after awaited futures complete.

### 14.5 Limitations

- **No tier-up**: There is currently no mechanism to JIT-compile a hot
  interpreted function into native code within an AOT runtime. The interpreter
  is the terminal execution tier.

- **No code sharing**: AOT code and bytecode use different object pool formats.
  A function cannot share code objects between the two modes.

- **Class hierarchy**: Dynamically added classes must be carefully integrated
  into the existing type system. Subtype checks and type tests must account for
  new classes. The `SubtypeTestCache` stubs handle this, but performance may
  degrade with many dynamically added types.

---

## 15. Summary of Key Source Locations

### Core bridging mechanism

| Component | File | Line(s) |
|---|---|---|
| `InterpretCall` stub (x64) | `runtime/vm/compiler/stub_code_compiler_x64.cc` | 3182-3268 |
| `InterpretCall` C function | `runtime/vm/runtime_entry.cc` | 4816-4856 |
| `InterpretCall` entry point | `runtime/vm/runtime_entry.cc` | 4859-4872 |
| `InvokeDartCodeFromBytecode` stub (x64) | `runtime/vm/compiler/stub_code_compiler_x64.cc` | 1748-1894 |
| Thread::interpret_call_entry_point_ | `runtime/vm/thread.h` | 266 |
| Stub declarations | `runtime/vm/stub_code_list.h` | 84 (InterpretCall), 93 (InvokeDartCodeFromBytecode) |

### Function / bytecode wiring

| Component | File | Line(s) |
|---|---|---|
| Function::AttachBytecode | `runtime/vm/object.cc` | 8555-8567 |
| Function::IsInterpreted | `runtime/vm/object.cc` | 8575-8577 |
| Function::HasBytecode | `runtime/vm/object.h` | 3303-3304 |
| Function bytecode accessors | `runtime/vm/object.h` | 3298-3310 |

### Interpreter engine

| Component | File | Line(s) |
|---|---|---|
| Interpreter class | `runtime/vm/interpreter.h` | 57-150 |
| Interpreter::Call (from C) | `runtime/vm/interpreter.cc` | 1739-1792 |
| Interpreter::Invoke (dispatch) | `runtime/vm/interpreter.cc` | 725-754 |
| Interpreter::InvokeBytecode | `runtime/vm/interpreter.cc` | 692-723 |
| Interpreter::InvokeCompiled | `runtime/vm/interpreter.cc` | 594-690 |
| Interpreter::Run (main loop) | `runtime/vm/interpreter.cc` | ~1936+ |
| Bytecode frame layout | `runtime/vm/stack_frame_kbc.h` | all |

### Bytecode loading

| Component | File | Line(s) |
|---|---|---|
| BytecodeLoader | `runtime/vm/bytecode_reader.h` | 21-87 |
| BytecodeReaderHelper | `runtime/vm/bytecode_reader.h` | 231-428 |
| loadDynamicModule native | `runtime/lib/object.cc` | 557-620 |
| loadDynamicModule Dart API | `sdk/lib/_internal/vm/lib/internal_patch.dart` | 468-477 |

### Dispatch table (AOT)

| Component | File | Line(s) |
|---|---|---|
| DispatchTable class | `runtime/vm/dispatch_table.h` | 14-73 |
| DispatchTableGenerator | `runtime/vm/compiler/aot/dispatch_table_generator.{h,cc}` | |
| DispatchTableCallInstr (IL) | `runtime/vm/compiler/backend/il.h` | 5101-5168 |
| Emission (x64) | `runtime/vm/compiler/backend/flow_graph_compiler_x64.cc` | 603-618 |
| Serialization | `runtime/vm/app_snapshot.cc` | 8904-9474 |
| Thread::dispatch_table_array_ | `runtime/vm/thread.h` | 853-914 |

### SwitchableCall resolution (AOT)

| Component | File | Line(s) |
|---|---|---|
| DRT_SwitchableCallMiss entry | `runtime/vm/runtime_entry.cc` | 3415-3448 |
| PatchableCallHandler::ResolveSwitchAndReturn | `runtime/vm/runtime_entry.cc` | 3260-3294 |
| PatchableCallHandler::ResolveTargetFunction | `runtime/vm/runtime_entry.cc` | 3194-3258 |
| PatchableCallHandler::HandleMissAOT | `runtime/vm/runtime_entry.cc` | 3298-3330 |
| PatchableCallHandler::DoUnlinkedCallAOT | `runtime/vm/runtime_entry.cc` | 2763-2805 |
| SwitchableCallMiss stub (x64) | `runtime/vm/compiler/stub_code_compiler_x64.cc` | 3830-3850 |
| SingleTargetCall stub (x64) | `runtime/vm/compiler/stub_code_compiler_x64.cc` | 3857-3895 |
| DRT_InterpretedInstanceCallMissHandler | `runtime/vm/runtime_entry.cc` | 3456-3486 |
| ExternalCall bytecode handler | `runtime/vm/interpreter.cc` | 2388-2436 |

### Build configuration

| Component | File | Line(s) |
|---|---|---|
| dart_dynamic_modules flag | `runtime/runtime_args.gni` | 79 |
| DART_DYNAMIC_MODULES define | `runtime/BUILD.gn` | (conditional) |

---

## Conclusion

The Dart VM already has a well-designed mixed-mode execution system gated behind
`DART_DYNAMIC_MODULES`. The key architectural insight is using the Function's
`code_` field as a mode discriminator: when it points to `InterpretCall`, all
existing dispatch mechanisms (static calls, dispatch table, switchable calls)
transparently route through the interpreter.

To implement an "extern function" model:

1. **Enable `DART_DYNAMIC_MODULES`** in the AOT build.
2. **Declare extern functions** in Dart source at compile time (even without
   implementations).
3. **Teach the precompiler** to emit these functions with InterpretCall as their
   code, and include appropriate dispatch table entries.
4. **At runtime**, use `BytecodeLoader` to load bytecode modules and attach
   bytecode to the placeholder functions via `AttachBytecode`.
5. **For new classes**, extend the ClassTable and dispatch table dynamically.

No fundamental architectural changes are needed. The existing infrastructure
handles stack interleaving, exception propagation, GC integration, and object
pool management across the boundary. The main work is in the precompiler (to
preserve extern function placeholders) and the bytecode loader (to resolve
extern references to existing Function objects).
