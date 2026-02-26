# Mixing AOT-Compiled Dart Code with Bytecode Interpreter

## Research Report: Architecture and Implementation Strategy

This report provides a comprehensive analysis of how the Dart VM supports (and can
further support) mixing ahead-of-time (AOT) compiled native code with bytecode-interpreted
code. The analysis is based on the existing `DART_DYNAMIC_MODULES` infrastructure already
present in the Dart VM.

---

## Table of Contents

1. [Executive Summary](#1-executive-summary)
2. [Existing Infrastructure: DART_DYNAMIC_MODULES](#2-existing-infrastructure-dart_dynamic_modules)
3. [Bootstrapping: Initializing a Hybrid AOT+Interpreter Runtime](#3-bootstrapping-initializing-a-hybrid-aotinterpreter-runtime)
4. [Transition Mechanism: AOT to Interpreted](#4-transition-mechanism-aot-to-interpreted)
5. [Transition Mechanism: Interpreted to AOT](#5-transition-mechanism-interpreted-to-aot)
6. [Extern Functions: Native Code Calling Not-Yet-Known Bytecode](#6-extern-functions-native-code-calling-not-yet-known-bytecode)
7. [Stack Frame Management in Mixed Mode](#7-stack-frame-management-in-mixed-mode)
8. [Dispatch Table and Call Site Management](#8-dispatch-table-and-call-site-management)
9. [Exception Handling Across Boundaries](#9-exception-handling-across-boundaries)
10. [GC and Stack Walking in Mixed Stacks](#10-gc-and-stack-walking-in-mixed-stacks)
11. [The Bytecode Format and Loading Pipeline](#11-the-bytecode-format-and-loading-pipeline)
12. [The Dynamic Interface Specification](#12-the-dynamic-interface-specification)
13. [Limitations and Open Issues](#13-limitations-and-open-issues)
14. [Key File Reference](#14-key-file-reference)

---

## 1. Executive Summary

The Dart VM already has a working (but off-by-default) infrastructure for mixing AOT
code with a bytecode interpreter, implemented under the `DART_DYNAMIC_MODULES` compile
flag (`runtime/runtime_args.gni:79`). When enabled, this flag activates:

- A full **KBC (Kernel Bytecode) interpreter** (`runtime/vm/interpreter.cc`)
- A **bytecode loader** that reads `.dbc` modules at runtime (`runtime/vm/bytecode_reader.cc`)
- An **InterpretCall stub** that serves as the trampoline from native code into the interpreter
- **InvokeDartCodeFromBytecode** stub that allows the interpreter to call back into native code
- Stack frame management that recognizes both native and interpreter frames
- Exception handling across the native/interpreted boundary

The key insight is: **a Function object's `code_` field can point to either native code
or the InterpretCall stub**. When it points to the InterpretCall stub, the function's
actual implementation resides as a `Bytecode` object in the `ic_data_array_or_bytecode_`
field. This allows complete transparency at call sites -- the caller does not need to know
whether the target is native or interpreted.

---

## 2. Existing Infrastructure: DART_DYNAMIC_MODULES

### Build Flag

The feature is gated behind a compile-time build flag:

```
# runtime/runtime_args.gni:78-79
# Whether to support dynamic loading and interpretation of Dart bytecode.
dart_dynamic_modules = false
```

When set to `true`, this defines the C++ preprocessor macro `DART_DYNAMIC_MODULES`
(`runtime/BUILD.gn:244-246`), which activates the interpreter, bytecode reader, and
all transition stubs.

### Core Components

| Component | File(s) | Purpose |
|-----------|---------|---------|
| Interpreter | `runtime/vm/interpreter.cc`, `.h` | KBC bytecode execution engine |
| Bytecode Reader | `runtime/vm/bytecode_reader.cc`, `.h` | Loads `.dbc` module files |
| KBC Constants | `runtime/vm/constants_kbc.h` | Bytecode instruction definitions |
| InterpretCall stub | `compiler/stub_code_compiler_{arch}.cc` | Trampoline: native -> interpreter |
| InvokeDartCodeFromBytecode | Same files | Trampoline: interpreter -> native |
| Stack frame (KBC) | `runtime/vm/stack_frame_kbc.h` | KBC frame layout constants |
| Bytecode object | `runtime/vm/object.h:7584+` | VM object holding bytecode instructions |
| Runtime entries | `runtime/vm/runtime_entry.cc:4816+` | InterpretCall C function |

### Dynamic Module Runner

A complete runner utility exists at `utils/dynamic_module_runner/`, built as an AOT
snapshot with a dynamic interface specification. It uses `pkg/dynamic_modules/` which
provides the Dart-level API:

```dart
// pkg/dynamic_modules/lib/dynamic_modules.dart
Future<Object?> loadModuleFromBytes(Uint8List bytes) =>
    internal.loadDynamicModule(bytes: bytes);
```

This calls into the VM via `@pragma("vm:external-name", "Internal_loadDynamicModule")`
(`sdk/lib/_internal/vm/lib/internal_patch.dart:476-477`), which triggers
`bytecode::BytecodeLoader::LoadBytecode()`.

---

## 3. Bootstrapping: Initializing a Hybrid AOT+Interpreter Runtime

### What Happens at VM Startup

1. **VM Initialization** (`runtime/vm/dart.cc:450+`): The VM snapshot is loaded. If it's
   an AOT snapshot (`Snapshot::kFullAOT`), all code from the snapshot is native-compiled.
   The VM initializes stub code including the `InterpretCall` and
   `InvokeDartCodeFromBytecode` stubs.

2. **Thread Setup** (`runtime/vm/thread.h:266`): Each Thread stores the
   `interpret_call_entry_point_` which points to the C function `InterpretCall`
   (`runtime/vm/runtime_entry.cc:4859-4872`). This entry point is stored in the Thread
   so the InterpretCall stub can load it efficiently from a known offset.

3. **Interpreter Creation**: When `DART_DYNAMIC_MODULES` is active, an `Interpreter`
   instance is created per isolate. It has its own stack (growing upward, separate from
   the native stack):
   ```
   Interpreter::Interpreter() {
     stack_ = new uintptr_t[stack_size];
     // stack_base_, overflow_stack_limit_, stack_limit_ set
   }
   ```

4. **Stub Registration** (`runtime/vm/stub_code_list.h`): The stubs `InterpretCall` and
   `InvokeDartCodeFromBytecode` are registered in the stub code list and generated
   during VM init.

### What Needs to Happen for a Custom Hybrid Runtime

To bootstrap a hybrid AOT+interpreter runtime:

1. Build the AOT snapshot with `dart_dynamic_modules = true` and a
   **dynamic interface specification** (see Section 12) that tells the precompiler which
   classes/methods may be extended or overridden by dynamically-loaded bytecode.

2. The precompiler retains extra metadata when `DART_DYNAMIC_MODULES` is enabled:
   - Abstract functions with entry point pragmas are kept (`precompiler.cc:1967-1974`)
   - Constant tables for exported classes are preserved (`precompiler.cc:2602-2608`)
   - `AsyncStarStreamController` is not dropped (`precompiler.cc:619-621`)

3. At runtime, after the AOT isolate is fully loaded, bytecode modules can be loaded
   via `Dart_LoadScriptFromBytecode()` or `Dart_LoadLibraryFromBytecode()`
   (`runtime/vm/dart_api_impl.cc:5619-5036`).

4. Loaded bytecode functions get the `InterpretCall` stub as their code, making them
   callable from native AOT code transparently.

---

## 4. Transition Mechanism: AOT to Interpreted

This is the primary transition direction: native AOT-compiled code calls a function
that is only available as bytecode.

### The InterpretCall Stub (the trampoline)

When a function is loaded from bytecode, `Function::AttachBytecode()` is called
(`runtime/vm/object.cc:8555-8567`):

```cpp
void Function::AttachBytecode(const Bytecode& value) const {
  // Store bytecode in the dual-purpose field
  untag()->set_ic_data_array_or_bytecode(value.ptr());
  // Set the code entry_point to InterpretCall stub
  SetInstructions(StubCode::InterpretCall());
}
```

After this, the function's `entry_point_` and `unchecked_entry_point_` both point to
the `InterpretCall` stub's entry. **Any native call site that calls this function through
normal dispatch (static call, virtual call, dispatch table) will transparently enter
the InterpretCall stub.**

### Stub Execution Flow (x64, `stub_code_compiler_x64.cc:3182-3268`)

The InterpretCall stub:

1. Enters a stub frame
2. Extracts the argument count from `ARGS_DESC_REG` (the arguments descriptor)
3. Adjusts for type arguments (adds 1 if type_args_len > 0)
4. Computes `argv` pointer: `argv = FP + argc * 8 + param_end_offset`
5. Negates `argc` (convention for indicating decreasing memory addresses)
6. Sets up the C calling convention:
   - Arg1: `FUNCTION_REG` (the Function object)
   - Arg2: `ARGS_DESC_REG` (arguments descriptor)
   - Arg3: negative argc
   - Arg4: argv pointer
   - Arg5: Thread pointer
7. Saves exit frame info (`top_exit_frame_info = RBP`)
8. Marks thread as exiting generated code
9. Calls the `interpret_call_entry_point` (loaded from Thread)
10. On return, restores VM tags and frame info
11. Returns the result in RAX

### The C++ InterpretCall Function (`runtime/vm/runtime_entry.cc:4816-4856`)

```cpp
extern "C" uword InterpretCall(uword function_in,
                                uword argdesc_in,
                                intptr_t argc,
                                ObjectPtr* argv,
                                Thread* thread) {
  Interpreter* interpreter = Interpreter::Current();
  ObjectPtr result = interpreter->Call(
      function, argdesc, argc, argv, Array::null(), thread);
  if (IsErrorClassId(result->GetClassId())) {
    // Propagate error
    Exceptions::PropagateError(error);
  }
  return static_cast<uword>(result);
}
```

### Key Behavioral Properties

- The caller does not know it is calling interpreted code. The call looks identical
  to calling any other function.
- Arguments are passed on the native stack in the standard ABI. The stub marshals
  them for the interpreter.
- The return value comes back in RAX as a tagged ObjectPtr.
- If the interpreter throws, the error propagates through `Exceptions::PropagateError`.

---

## 5. Transition Mechanism: Interpreted to AOT

When the interpreter needs to call a function that has native (AOT) code, it uses
`Interpreter::InvokeCompiled()` (`runtime/vm/interpreter.cc:594-654`).

### The InvokeCompiled Method

```cpp
bool Interpreter::InvokeCompiled(Thread* thread,
                                  FunctionPtr function,
                                  ObjectPtr* call_base, ...) {
  invokestub entrypoint = reinterpret_cast<invokestub>(
      StubCode::InvokeDartCodeFromBytecode().EntryPoint());

  Exit(thread, *FP, call_top + 1, *pc);
  {
    InterpreterSetjmpBuffer buffer(this);
    if (!DART_SETJMP(buffer.buffer_)) {
      result = entrypoint(
          function->untag()->entry_point_,  // or code() in JIT
          argdesc_,
          call_base,
          thread);
      Unexit(thread);
    } else {
      return false;  // Exception occurred
    }
  }
  // Store result and continue interpreting
}
```

### Key Points

- `Exit()` saves the interpreter's frame state so the GC can walk the interpreter stack.
- The `InterpreterSetjmpBuffer` captures a longjmp point. If the native code throws,
  the exception handler will longjmp back to this point.
- `InvokeDartCodeFromBytecode` is a regular VM entry stub (similar to `InvokeDartCode`)
  but adapted for the interpreter's calling convention. It transitions from
  "interpreter state" to "generated code state."
- On return, `Unexit()` restores the interpreter's frame pointers.

### The Invoke Dispatcher (`interpreter.cc:725-754`)

The interpreter's general `Invoke()` method handles the dispatch decision:

```cpp
bool Interpreter::Invoke(...) {
  FunctionPtr function = FrameFunction(callee_fp);
  for (;;) {
    if (Function::IsInterpreted(function)) {
      return InvokeBytecode(...);   // Stay in interpreter
    } else if (Function::HasCode(function)) {
      return InvokeCompiled(...);   // Transition to native
    }
    // If neither, try to compile the function
    // (calls DRT_CompileFunction runtime entry)
  }
}
```

This is the central dispatch point. It checks the function's state and routes
accordingly. If a function has neither bytecode nor code, it triggers compilation.

---

## 6. Extern Functions: Native Code Calling Not-Yet-Known Bytecode

This is the most interesting scenario: AOT-compiled code expects a method to exist
that will only be provided later via bytecode (an "extern" pattern).

### How It Works Today

The Dart VM already supports this pattern through several mechanisms:

#### A. The LazyCompile Stub as Placeholder

In AOT mode, functions that are not yet compiled have their `code_` field set to the
`LazyCompile` stub (`runtime/vm/app_snapshot.cc:1830,8165`):

```
// Code index 0 is reserved for the LazyCompile stub
```

However, in AOT mode, `LazyCompile` usually triggers a fatal error since all code
should be precompiled. With `DART_DYNAMIC_MODULES`, a different path is possible.

#### B. Virtual/Interface Dispatch via Dispatch Table

In AOT mode, virtual method calls go through the **dispatch table**
(`runtime/vm/dispatch_table.h`). Each selector (method name) gets an offset, and
each class has entries at that offset pointing to the implementation.

For the hybrid case, the dispatch table entry for a not-yet-loaded method would need
to point to a **trampoline** that either:
1. Resolves and loads the bytecode module, then calls the interpreted function
2. Triggers a "method not found" error with an informative message

#### C. The Dynamic Interface Specification

The precompiler uses a **dynamic interface YAML file** to know which methods might be
overridden or called from dynamically-loaded code. This specification has three sections:

```yaml
# utils/dynamic_module_runner/dynamic_interface.yaml
extendable:
  - library: 'dart:core'
  # ... classes that can be subclassed by bytecode modules

can-be-overridden:
  - library: 'dart:core'
  # ... methods that bytecode modules may override

callable:
  - library: 'dart:core'
  # ... methods that bytecode modules may call
```

The `dynamic_interface_annotator.dart` (`pkg/vm/lib/transformations/`) processes this
and adds pragmas to the kernel:
- `@pragma('dyn-module:extendable')` on classes
- `@pragma('dyn-module:can-be-overridden')` on methods
- `@pragma('dyn-module:callable')` on callable targets

The precompiler then ensures that:
- Extendable classes keep their vtable structure open
- Overridable methods retain their dispatch table slots
- Callable methods are not tree-shaken

#### D. Implementing Extern Functions

For a function that native code expects but bytecode will provide, the approach is:

1. **At compile time (precompiler)**: The function is declared in the static program
   (possibly as an abstract method or with a stub body). The dynamic interface
   specification marks it as `can-be-overridden`. The precompiler assigns it a dispatch
   table slot but may use the `InterpretCall` stub as its initial entry.

2. **At runtime (bytecode loading)**: When the bytecode module is loaded via
   `BytecodeLoader::LoadBytecode()`, it registers new classes and methods. The bytecode
   reader calls `Function::AttachBytecode()` on the loaded function, which sets its
   entry point to the `InterpretCall` stub and stores the `Bytecode` object.

3. **At call time**: When native code calls the function (through dispatch table or
   direct call), it transparently enters the `InterpretCall` stub, which invokes the
   interpreter with the bytecode.

#### E. The Switchable Call Mechanism

For polymorphic call sites in AOT mode, the VM uses **switchable calls**
(`runtime/vm/runtime_entry.cc:2656-3294`). The call site transitions through states:

```
UnlinkedCall -> Monomorphic -> SingleTargetCache -> ICData -> MegamorphicCache
```

When a bytecode-provided method override is loaded, the existing monomorphic or
polymorphic call caches may become stale. The VM would need to:
1. Invalidate affected call site caches
2. Allow the `PatchableCallHandler` to resolve to interpreter targets
3. Update `MegamorphicCache` entries to point to `InterpretCall`-backed functions

The mechanism already handles this conceptually: `MegamorphicCache` entries store
`FunctionPtr` targets, and a function pointing to `InterpretCall` is a valid target.

#### F. ExternalCall Bytecode Instruction

The interpreter has an `ExternalCall` instruction (`interpreter.cc:2388-2436`) for
calling native (C++) functions from bytecode. If the native function hasn't been
resolved yet, it calls `DRT_ResolveExternalCall` (`runtime_entry.cc:1148-1188`) which:

1. Gets the function's `native_name()`
2. Uses the library's `native_entry_resolver()` callback
3. Resolves the native function pointer
4. Patches the constant pool so future calls go directly to the resolved function

This is the inverse pattern: bytecode calling into native code for functions the
bytecode doesn't implement itself.

---

## 7. Stack Frame Management in Mixed Mode

### Two Separate Stacks

The interpreter maintains its own stack (growing upward) separate from the native stack
(growing downward). The interpreter stack holds KBC frames:

```
// runtime/vm/stack_frame_kbc.h
kKBCDartFrameFixedSize = 4  // Function, Bytecode, caller PC, caller FP
kKBCFunctionSlotFromFp = -4
kKBCPcMarkerSlotFromFp = -3
kKBCSavedCallerPcSlotFromFp = -2
kKBCSavedCallerFpSlotFromFp = -1
```

While native frames use the architecture-specific layout (e.g., x64):

```
// runtime/vm/stack_frame_x64.h
kDartFrameFixedSize = 4
kPcMarkerSlotFromFp = -1
kSavedCallerPpSlotFromFp = -2
kSavedCallerFpSlotFromFp = 0
kSavedCallerPcSlotFromFp = 1
```

### Frame Detection

The `StackFrameIterator` detects whether a frame is an interpreter frame using
`CheckIfInterpreted()` (`runtime/vm/stack_frame.cc:609-611`):

```cpp
frames_.CheckIfInterpreted(exit_marker);
// This calls: Interpreter::Current()->HasFrame(exit_marker)
// which checks: frame >= stack_base_ && frame < stack_limit_
```

This determines which frame layout to use for stack walking, GC, and exception handling.

### Interleaved Frames

A typical mixed stack looks like:

```
[Native frame - main()]
  -> calls interpreted function via InterpretCall stub
[Exit frame - saves native state]
[Interpreter entry frame]
  [KBC frame - interpreted function]
    -> calls native function via InvokeCompiled/InvokeDartCodeFromBytecode
  [Interpreter exit frame - saves interpreter state]
[Entry frame - InvokeDartCodeFromBytecode stub]
  [Native frame - native function]
    -> ...
```

The alternation between native and interpreter frames is tracked through the chain of
exit/entry frames, with each transition saving the state needed to resume.

---

## 8. Dispatch Table and Call Site Management

### AOT Dispatch Table

The AOT dispatch table (`runtime/vm/dispatch_table.h`) is a flat array of code entry
points, indexed by `class_id * selector_count + selector_offset`. For dynamically-loaded
classes, the table would need to be extended.

The `DispatchTableGenerator` (`compiler/aot/dispatch_table_generator.cc`) assigns
offsets to selectors during precompilation. To support bytecode-provided method
overrides:

1. The selector offsets for `can-be-overridden` methods are reserved during AOT compilation
2. At runtime, when a bytecode module provides an override, the dispatch table entry
   for the new class/selector combination is patched to point to the `InterpretCall`
   stub's entry point (or a per-function entry if needed)

### Call Site Adaptation

For **static calls** to functions that might be bytecode-provided:
- The call site targets `Function.entry_point_` which points to `InterpretCall` stub
- No call site patching needed; the function object itself is the indirection

For **virtual/interface calls** through dispatch table:
- The dispatch table slot for the new class+selector is updated
- The megamorphic cache is extended with the new (class_id, function) entry

For **dynamic calls** (via `noSuchMethod` or dynamic dispatch):
- The resolver (`runtime/vm/resolver.cc`) finds the function in the class hierarchy
- If the function is bytecode-backed, it returns it normally
- The call proceeds through `InterpretCall`

---

## 9. Exception Handling Across Boundaries

### Native -> Interpreted

When native code calls an interpreted function via the InterpretCall stub and the
interpreter throws:

1. The interpreter catches the exception internally
2. `InterpretCall` C function checks for error return (`runtime_entry.cc:4844-4854`):
   ```cpp
   if (IsErrorClassId(result->GetClassId())) {
     TransitionGeneratedToVM transition(thread);
     Exceptions::PropagateError(error);
   }
   ```
3. The error propagates through the normal native exception handling mechanism

### Interpreted -> Native

When the interpreter calls native code via `InvokeCompiled` and the native code throws:

1. The exception triggers a longjmp via the `InterpreterSetjmpBuffer`
2. The `DART_SETJMP/DART_LONGJMP` mechanism restores the interpreter's frame
3. The interpreter then handles the exception through its own exception table

### Exception Handler Lookup in Mixed Stacks

The `StackFrame::FindExceptionHandler()` method (`stack_frame.cc:470-544`) handles
both frame types:

- For interpreted frames: loads bytecode via `LookupDartBytecode()`, gets
  `exception_handlers` from the bytecode, calls `bytecode.GetTryIndexAtPc(pc())`
- For native frames: loads code via `LookupDartCode()`, gets handlers from code
  object, uses `pc_descriptors` to find the try_index

---

## 10. GC and Stack Walking in Mixed Stacks

### Object Pointer Visiting

The GC must find all live object pointers on both stacks. The
`StackFrame::VisitObjectPointers()` method handles this differently for each frame type:

- **Native frames**: Uses compressed stack maps from the Code object to identify
  which stack slots contain object pointers
- **Interpreter frames**: Visits the fixed slots (`kKBCFirstObjectSlotFromFp` to
  `kKBCLastFixedObjectSlotFromFp`) plus the entire local/operand area

The interpreter also implements `VisitObjectPointers()` (`interpreter.h:119`) to
visit its own state (object pool, arguments descriptor, special slots).

### Stack Walking

The `StackFrameIterator` alternates between native and interpreter frame sets.
At each exit frame, it checks `CheckIfInterpreted()` to determine which frame
layout to use for the next set of frames. The entry/exit frame chain maintains
the interleaving:

```
Native exit -> Interpreter entry -> KBC frames -> Interpreter exit
  -> Native entry -> Native frames -> Native exit -> ...
```

---

## 11. The Bytecode Format and Loading Pipeline

### Bytecode File Format (`pkg/dart2bytecode/docs/bytecode.md`)

The bytecode format uses magic number `0x44424333` ('DBC3'), format version 1.
Structure:

```
BytecodeFile {
  UInt32 magic, formatVersion
  Section[] sections    // descriptors for all sections below
  StringTable           // de-duplicated string data
  ObjectTable           // shared objects and constants
  EntryPoint            // main function offset
  LibraryIndex[]        // URI -> offset mapping
  LibraryDeclaration[]  // library metadata
  ClassDeclaration[]    // class metadata
  Members[]             // field and function declarations
  Code[]                // bytecode instructions + constant pools
  SourcePositions[]     // PC -> source mapping
  SourceFile[]          // source file content
  ...
}
```

### Loading Pipeline

1. **API Entry**: `Dart_LoadLibraryFromBytecode()` or `Internal_loadDynamicModule`
2. **Bytecode Loading**: `BytecodeLoader::LoadBytecode()` (`bytecode_reader.cc`)
   - Reads the bytecode component header
   - Reads library declarations, registering them with the VM
   - Reads class declarations, creating Class objects
   - Reads member declarations, creating Function and Field objects
   - Reads code sections, creating Bytecode objects
3. **Function Setup**: For each function with bytecode:
   - `Function::AttachBytecode(bytecode)` is called
   - This sets `code_ = InterpretCall` stub
   - And stores bytecode in `ic_data_array_or_bytecode_`
4. **Class Finalization**: `ClassFinalizer` processes new classes, integrating them
   into the type system and hierarchy

### Instruction Set

The KBC instruction set (`runtime/vm/constants_kbc.h:37-246`) includes ~100 opcodes:

- **Function entry**: `Entry`, `EntryOptional`, `EntrySuspendable`
- **Stack ops**: `PushConstant`, `PushNull`, `Push`, `Pop`, `Drop1`
- **Calls**: `DirectCall`, `InterfaceCall`, `DynamicCall`, `UncheckedDirectCall`,
  `ExternalCall` (for native calls), `UncheckedClosureCall`
- **Control flow**: `Jump`, `JumpIfTrue/False/Null/NotNull/EqStrict/NeStrict`
- **Object allocation**: `Allocate`, `AllocateT`, `CreateArrayTOS`, `AllocateClosure`,
  `AllocateContext`, `AllocateRecord`
- **Field access**: `LoadFieldTOS`, `StoreFieldTOS`, `LoadStatic`, `StoreStaticTOS`
- **Type ops**: `AssertAssignable`, `InstantiateType`, `CheckFunctionTypeArgs`
- **Optimized arithmetic**: `AddInt`, `SubInt`, `MulInt`, `CompareIntEq/Gt/Lt`, etc.
- **Async**: `Suspend` for saving frame state

---

## 12. The Dynamic Interface Specification

### Purpose

The dynamic interface specification tells the AOT precompiler which parts of the
statically-compiled program may interact with dynamically-loaded bytecode modules.
Without this, the precompiler aggressively tree-shakes and devirtualizes, which would
break dynamic module loading.

### Categories

The specification has three categories:

1. **`extendable`**: Classes that bytecode modules may subclass. The precompiler must:
   - Keep the class's vtable structure
   - Not seal the class hierarchy for devirtualization
   - Retain constructors that might be super-called
   - Annotated with `@pragma('dyn-module:extendable')`

2. **`can-be-overridden`**: Methods that bytecode modules may override. The precompiler must:
   - Keep dispatch table slots for these methods
   - Not inline these methods (since the implementation may change)
   - Retain the method's signature metadata
   - Copy annotations to mixin applications
   - Annotated with `@pragma('dyn-module:can-be-overridden')`

3. **`callable`**: Methods that bytecode modules may call. The precompiler must:
   - Not tree-shake these methods
   - Keep their code in the AOT snapshot
   - Annotated with `@pragma('dyn-module:callable')`

### Processing Pipeline

1. The dynamic interface YAML is passed to the kernel compiler via
   `--dynamic-interface=` flag
2. `dynamic_interface_annotator.dart` processes the spec and annotates the kernel AST
3. The precompiler reads these annotations during `AddAnnotatedRoots()` and
   `TraceForRetainedFunctions()`
4. Annotated functions/classes survive tree-shaking and devirtualization

---

## 13. Limitations and Open Issues

### Current Limitations

1. **Separate stacks**: The interpreter has its own stack separate from the native stack.
   Transitions between the two have overhead (saving/restoring state, computing argv).
   Sharing a single stack would reduce transition cost.

2. **No tier-up**: Currently, a function is either interpreted or compiled. There's no
   mechanism to JIT-compile a "hot" interpreted function in AOT mode.

3. **Dispatch table extension**: Adding new classes at runtime requires extending the
   dispatch table. The current table is fixed-size. Dynamic extension would need
   reallocation or a secondary lookup table.

4. **Monomorphic call invalidation**: When a bytecode module provides an override for
   a method that has been monomorphically cached at a call site, the cache needs
   invalidation. This is not fully handled for all call site states.

5. **Architecture support**: The InterpretCall and InvokeDartCodeFromBytecode stubs
   are only implemented for x64, ARM64, ARM, and IA32. RISC-V has stubs but they
   are not implemented (`Stop("Not implemented on RISC-V.")`).

6. **No incremental bytecode**: The bytecode format loads entire modules. There's no
   support for loading individual functions on demand.

### Design Considerations for Extension

- **Lazy resolution pattern**: For extern functions not yet loaded, a
  "not yet resolved" stub could be installed that either loads the required module
  or throws a clear error. This parallels the `LazyCompile` stub pattern.

- **Module dependency tracking**: When module A depends on functions from module B,
  the VM needs to track these dependencies and load B before A (or provide stubs).

- **Type system coherence**: Bytecode modules must be type-checked against the static
  program's types. The `ClassFinalizer` handles this but may need extension for
  complex generic hierarchies.

---

## 14. Key File Reference

### Core Hybrid Mode Infrastructure

| File | Key Lines | Description |
|------|-----------|-------------|
| `runtime/runtime_args.gni` | 78-79 | `dart_dynamic_modules` build flag |
| `runtime/BUILD.gn` | 244-246 | Defines `DART_DYNAMIC_MODULES` preprocessor macro |
| `runtime/vm/interpreter.h` | 57-283 | Interpreter class definition |
| `runtime/vm/interpreter.cc` | 594-754 | Invoke, InvokeBytecode, InvokeCompiled |
| `runtime/vm/interpreter.cc` | 1891-2436 | Main dispatch loop and bytecodes |
| `runtime/vm/constants_kbc.h` | 37-246 | Bytecode instruction set definition |
| `runtime/vm/bytecode_reader.h` | 21-87 | BytecodeLoader class |
| `runtime/vm/bytecode_reader.cc` | - | Bytecode reading and loading |
| `runtime/vm/object.h` | 7584-7713 | Bytecode object class |
| `runtime/vm/object.h` | 3303-3308 | Function::HasBytecode, IsInterpreted |
| `runtime/vm/object.cc` | 8555-8577 | Function::AttachBytecode, ClearBytecode, IsInterpreted |
| `pkg/dart2bytecode/docs/bytecode.md` | - | Bytecode binary format spec |

### Transition Stubs

| File | Key Lines | Description |
|------|-----------|-------------|
| `compiler/stub_code_compiler_x64.cc` | 3163-3177 | LazyCompile stub |
| `compiler/stub_code_compiler_x64.cc` | 3182-3268 | InterpretCall stub |
| `compiler/stub_code_compiler_x64.cc` | 1748-1850 | InvokeDartCodeFromBytecode stub |
| `compiler/stub_code_compiler_arm64.cc` | 3219-3295 | Same for ARM64 |
| `runtime/vm/runtime_entry.cc` | 4816-4872 | InterpretCall C function |
| `runtime/vm/runtime_entry.cc` | 4879-4920 | ResumeInterpreter runtime entry |

### Stack Frames and Walking

| File | Key Lines | Description |
|------|-----------|-------------|
| `runtime/vm/stack_frame_kbc.h` | 38-56 | KBC frame layout constants |
| `runtime/vm/stack_frame.cc` | 604-663 | CheckIfInterpreted in iterator |
| `runtime/vm/stack_frame.cc` | 253-389 | VisitObjectPointers (both frame types) |
| `runtime/vm/stack_frame.cc` | 470-544 | FindExceptionHandler (both types) |

### Dispatch and Call Sites

| File | Key Lines | Description |
|------|-----------|-------------|
| `runtime/vm/dispatch_table.h` | 14-73 | Dispatch table structure |
| `compiler/aot/dispatch_table_generator.cc` | 405-634 | Dispatch table generation |
| `runtime/vm/runtime_entry.cc` | 2656-3294 | PatchableCallHandler (switchable calls) |
| `runtime/vm/code_patcher.h` | 76-82 | Call site patching API |
| `compiler/aot/precompiler.cc` | 451-620 | DoCompileAll with dynamic module support |

### Dynamic Module API and Loading

| File | Key Lines | Description |
|------|-----------|-------------|
| `runtime/vm/dart_api_impl.cc` | 5619-5657 | Dart_LoadScriptFromBytecode |
| `runtime/vm/dart_api_impl.cc` | 6016-6036 | Dart_LoadLibraryFromBytecode |
| `runtime/lib/object.cc` | 557-607 | Internal_loadDynamicModule native |
| `sdk/lib/_internal/vm/lib/internal_patch.dart` | 468-477 | loadDynamicModule Dart API |
| `pkg/dynamic_modules/lib/dynamic_modules.dart` | - | Public Dart API |
| `pkg/vm/lib/transformations/dynamic_interface_annotator.dart` | - | Annotation processing |

### Object Model

| File | Key Lines | Description |
|------|-----------|-------------|
| `runtime/vm/object.h` | 3050-3346 | Function class (entry points, code) |
| `runtime/vm/raw_object.h` | 1369-1570 | UntaggedFunction raw fields |
| `runtime/vm/raw_object.h` | 1991-2105 | UntaggedCode raw fields |
| `runtime/vm/object.h` | 6927-7200 | Code class (4 entry point kinds) |
| `runtime/vm/object.h` | 5853-6013 | Instructions class (entry offsets) |
| `runtime/vm/thread.h` | 266 | interpret_call_entry_point_ field |
| `runtime/vm/dart_entry.cc` | 131-178 | InvokeFunction (interpreted path) |
