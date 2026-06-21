---
layout: post
title:  "Fuzzing macOS and iOS Kernels with DarwinKit"
date:   2026-06-21 12:00:00
categories: blog
---

Fuzzing closed-source operating system kernels has historically been a complex and resource-intensive endeavor. Security researchers targeting Apple's platforms often faced a choice between slow software emulation or expensive physical hardware setups with complex cabling and fragile jailbreak scripts.

To bridge this gap, we developed **DarwinKit**—our security research, reverse engineering, and binary instrumentation toolkit designed specifically for the XNU kernel family. DarwinKit bridges the Apple and Google ecosystems by integrating Google's Bazel build system, Google FuzzTest, and AFL++'s LibAFL framework. 

Below is a technical deep dive into how DarwinKit performs high-performance local kernel fuzzing: starting with our mature pipeline on macOS, and then detailing how we extended this architecture to support virtualized iOS targets.

---

# Part 1: macOS Kernel Fuzzing & Instrumentation

DarwinKit's macOS support is mature, providing symbols parsing, memory patching, and coverage-guided fuzzing directly on local hardware.

## 1. Building the macOS Components
DarwinKit uses Bazel to compile kernel-mode and user-mode binaries for both `x86_64` and `arm64` architectures:

```bash
# Build the native macOS Kernel Extension (kext)
bazel build --macos_cpus=arm64 --action_env=PATH //macos:DarwinKit

# Build the userspace CLI tool (darwinkit_tool)
bazel build --macos_cpus=arm64 --action_env=PATH //macos:darwinkit_tool

# Build the userspace library (DarwinKit_user)
bazel build --macos_cpus=arm64 --action_env=PATH //macos:DarwinKit_user
```

## 2. In-Place KernelCache Patching with `kc_patcher`
To collect code coverage during fuzzing, we must instrument the basic blocks of the target kernel and kext binaries (e.g., `com.apple.kernel` or `com.apple.iokit.IOSurface`).

Legacy tools like Pishi rely on Ghidra's headless analyzer or heavyweight Java APIs to load the massive 30-100MB prelinked KernelCache, parse symbols, generate a Control Flow Graph (CFG), and identify basic blocks. This analysis is extremely slow—often taking **2 to 4 hours**.

We solved this bottleneck by building `kc_patcher`, a lightweight Rust-based utility that performs the entire pipeline in **less than 1 second**:
* **Direct Memory Mapping**: Loads the kernelcache directly into a contiguous memory buffer and parses it in-place via a custom Mach-O parser (`macho.rs`).
* **Fast Custom Disassembler**: Leverages a fast AArch64 decoding module (`disasm.rs`) to scan only target `__TEXT_EXEC` segments.
* **Rapid CFG Construction**: Builds the control flow graph using a single-pass linear scan to identify block leaders, followed by a breadth-first search (BFS) traversal to purge unreachable basic blocks.
* **dSYM-Guided Instrumentation**: When provided with a Kernel Debug Kit (KDK) dSYM path, `kc_patcher` parses DWARF debug info to locate exact function boundaries and match them against target symbols in the kernelcache, avoiding heuristic search bugs.

```bash
# Instruments com.apple.kernel in the boot.kc image using KDK dSYMs
./target/release/kc_patcher instrument \
  --kernelcache boot.kc \
  --output boot.kc \
  --instrument com.apple.kernel \
  --dsym /Library/Developer/KDKs/KDK_26.2_25C56.kdk/System/Library/Kernels/kernel.release.t8130.dSYM
```

## 3. Booting the Custom Kernel Cache
To load our instrumented kernel cache on a local macOS machine:
1. Reboot the Mac and enter **Recovery Mode**.
2. Open the terminal and disable System Integrity Protection (SIP) and user-approved kext restrictions:
   ```bash
   csrutil disable
   ```
3. Reboot normally, run `prepare_kernel.sh` to extract the built kext, generate `boot.kc`, and configure `kmutil` to use it:
   ```bash
   sudo kmutil configure-boot -v / -c $(pwd)/boot.kc
   ```
4. Reboot the system to boot with the custom instrumented kernel cache.

## 4. Running Userspace GoogleFuzzTest Campaigns
For userspace targets (like parser components extracted from the kernel), DarwinKit integrates Google's **FuzzTest** engine:

```bash
bazel run --config=fuzztest //tests:macho_test -- --fuzz=MachOTest.MachOFuzzWithCorpus
```

---

# Part 2: Extending support to iOS Kernel Fuzzing

While macOS runs on local host hardware, fuzzing the iOS kernel requires adapting DarwinKit for virtualization and dealing with iOS-specific security constraints.

## 1. Adapting the Bazel Toolchain for iOS
Because upstream Bazel does not natively support building native iOS Kernel Extensions (`.kext`), we imported custom overrides in our `MODULE.bazel` configuration:

```python
# MODULE.bazel overrides for iOS kernel builds
git_override(
    module_name = "rules_apple",
    commit = "f5be07ba4d38408d6d1747337e4dd1f598a6d34d",
    remote = "https://github.com/bazelbuild/rules_apple.git",
)

git_override(
    module_name = "apple_support",
    commit = "8c46f76e85bf8aee9562553577da0ae953c389d8",
    remote = "https://github.com/bazelbuild/apple_support.git",
)
```
These forks expose the `ios_kernel_extension()` macro, allowing us to compile native `.kext` files for the target iOS architecture.

## 2. Hijacking existing iOS Kexts
On iOS, the prelinked KernelCache expects a static, signed fileset layout. Adding new load commands to insert our kext is rejected by the iOS bootloader. 

To bypass this, `kc_patcher` performs **kext hijacking**:
* It targets a large, non-essential system fileset kext in the prelinked kernel cache (e.g., `com.apple.driver.AppleM2ScalerCSCDriver` or `com.apple.driver.FairPlayIOKit`).
* It overwrites the victim's executable segments (`__TEXT_EXEC`) with DarwinKit's compiled payload.
* It mutates the plist dictionary inside the `__PRELINK_INFO` XML segment, updating the bundle identifier and sizes, and points the `kmod_info` entry points to DarwinKit.
* It renames the `LC_FILESET_ENTRY` to `com.YungRaj.DarwinKit` to satisfy linker resolution.

## 3. Overcoming the Zero Writable Data Segment Rule & Chained Fixups

One of the most complex engineering challenges we encountered when porting DarwinKit to iOS was the complete ban on writable data segments. The iOS kernel extension loader enforces extremely strict code signature verification. Specifically, it rejects any injected or modified kext that contains writable data segments (`__DATA`) or read-only constant tables (`__DATA_CONST`).

### The Chained Fixups Problem

The technical reason we must avoid all data segments centers around **dyld chained fixups** in the iOS prelinked KernelCache. 

When XNU loads a prelinked KernelCache, it must adjust all absolute pointer values in memory to account for kernel address space layout randomization (KASLR). In modern iOS versions, Apple replaced traditional Mach-O relocation tables with dyld chained fixups. In this scheme, pointer locations are chained together in memory: each pointer location contains a small offset header pointing to the next fixup location. 

If our injected kext references global variables, read-only string tables, or virtual tables (vtables), these absolute references generate dynamic rebases and binds. These binds require corresponding metadata entries in the KernelCache's chained fixup structures. However, since the prelinked KernelCache is statically signed and laid out, we cannot expand or modify the chained fixup tables without corrupting the fixup chain or triggering code signature verification failures during the early boot sequence. The moment the kernel tries to process an invalid or modified fixup chain, the device instantly panics and boot-loops.

To solve this, we must ensure DarwinKit operates with **zero global memory usage** and **zero data segments**. The compiled binary must place all static data directly into the executable text segment (`__TEXT`) or construct it dynamically on the stack at runtime.

### Eliminating Globals and the `inline_str!` Macro

To satisfy the zero-data segment constraint, the DarwinKit iOS kernel module compiles with a custom Rust target JSON (`arm64e-kernel-ios.json`) under the `no_data_segment` feature flag. This forces the compiler to omit default data segments and restricts us to pure position-independent code.

This architecture requires the following compiler and implementation rules:

1. **Zero Global Writable State**: Every piece of fuzzing state—including the LibAFL engine, mutators, inputs, and executors—must be allocated entirely on the stack or dynamically allocated at runtime using resolved `IOMalloc` kernel heap pointers. We cannot use static `lazy_static` or global `Mutex` buffers.
2. **Dynamic String Declarations via `inline_str!`**: Standard string literals (like `"hello"`) are compiled by Rust/C++ into a global read-only data section (`__TEXT,__cstring` or `__DATA_CONST`), requiring a pointer relocation. To bypass this, we define all strings using a custom assembly macro, `inline_str!`. 

Here is the exact implementation of the `inline_str!` macro in DarwinKit's core library:

{% raw %}
```rust
#[macro_export]
macro_rules! inline_str {
    ($s:expr, $len:expr) => {{
        let ptr: *const u8;
        unsafe {
            core::arch::asm!(
                "adr {0}, 1f",                  // Load PC-relative address of label 1:
                "b 2f",                         // Branch forward to skip the string bytes
                concat!("1: .ascii \"", $s, "\""), // Place raw ASCII string bytes in instruction stream
                ".p2align 2",                   // Align to instruction boundary
                "2:",                           // Resume execution target
                out(reg) ptr,
            );
            core::str::from_utf8_unchecked(core::slice::from_raw_parts(ptr, $len))
        }
    }};
}
```
{% endraw %}

By embedding the raw ASCII bytes directly within the executable instructions (`__TEXT_EXEC` segment) and resolving its address via PC-relative assembly instructions (`adr`), the string is loaded onto the stack at runtime. This requires **zero relocation entries or chained fixup structures**, keeping the data segment completely empty.

We also apply several other strict rules across the library:

| Optimization / Rule | Implementation Details |
| :--- | :--- |
| **No vtables (Static Dispatch)** | Dynamic trait dispatch (`dyn Trait`) generates vtables in the `__DATA` segment. We replaced all trait objects with monomorphized static types and generic interfaces. |
| **ASCII-only Processing** | Standard case-folding and Unicode lookup tables (e.g., `LOWERCASE_LUT`) generate static lookup tables. We replaced standard string operations with manual byte-level ASCII checks. |
| **Scaled Integer Arithmetic** | Float-to-string conversions normally pull in massive lookup tables (`core::num::flt2dec`). We bypassed this by scaling decimals to basis points and formatting them manually onto stack-allocated buffers. |
| **Immediate Abort Panics** | We compile with `-C panic=immediate-abort` to completely strip panic formatting string metadata (such as source filenames and line numbers) which would otherwise reside in global memory. |
| **No Hash Tables** | Standard hash maps rely on global state or static bucket allocations. We replaced `HashSet` and `HashMap` with stack-constructed `BTreeSet` and `BTreeMap` structures. |

We run post-build verification tests (`check_zero_data.sh`) that parse the output Mach-O headers using `objdump -h` and fail the build if `__DATA` or `__DATA_CONST` segments are detected.


## 4. In-Kernel Fuzzing & The Stack Pivot
To avoid the massive performance degradation of hypercall boundaries (VM exits), DarwinKit runs a highly customized `no_std` version of the **LibAFL** fuzzing framework directly inside the guest kernel space.

However, XNU kernel stacks are extremely small (typically 16KB). A complex fuzzing engine executing mutations, generating test inputs, and running serialization would easily trigger a kernel stack overflow panic.

To resolve this, we implement a custom stack pivot (`pivot.rs`) in AArch64 assembly. When user space triggers a fuzzing campaign, DarwinKit allocates a 1MB stack using the kernel's `IOMalloc` and pivots the stack pointer (`SP`) and frame pointer (`FP`) to this new region:

```rust
// AArch64 Stack Pivot assembly representation
core::arch::asm!(
    "mov sp, {new_sp}",              // Set SP to top of 1MB allocated stack
    "mov x29, {new_sp}",             // Set FP (x29) to match
    "ldr x30, [{new_sp}, #8]",       // Load target fuzzer return address
    "mov x0, {bitmap}",              // Pass coverage bitmap address as argument
    "blr {fuzzer}",                  // Branch and link to LibAFL fuzzer entry
    // Fuzzer execution completes, restore host stack:
    "mov sp, {in_sp}",
    "mov x29, {in_fp}",
    "mov lr, {in_lr}",
    new_sp = in(reg) stack_top,
    fuzzer = in(reg) fuzzer_fn,
    bitmap = in(reg) bitmap_ptr,
    in_sp = in(reg) old_sp,
    in_fp = in(reg) old_fp,
    in_lr = in(reg) old_lr,
);
```

### Coverage Tracing
To collect basic-block level coverage, `kc_patcher` patches target basic blocks to branch to pre-allocated assembly thunks (`instrument_thunks`). 

The thunks preserve NZCV flags and registers before calling `sanitizer_cov_trace_pc(kext_id, address)` to write directly into our 64KB coverage bitmap.

## 5. Local Virtualization with `vphone-cli`
With the KernelCache patched and instrumented, we utilize `vphone-cli` on Apple Silicon Macs to run the guest environment. Because the host Mac and guest iOS target share the ARM64 architecture, the OS executes natively via the hypervisor at hardware speed.

We boot the virtualized iOS environment with our target target dependencies loaded:

```bash
cd modules/vphone-cli
export BAZEL_TARGET=//ios:DarwinKit
JB=1 make setup_machine
```

Once booted, userspace components communicate directly with the `/dev/darwinkit` device. If a crash or memory corruption occurs during fuzzing, `vphone-cli` restores a clean VM snapshot in milliseconds, ensuring a highly deterministic and high-throughput kernel fuzzing loop.
