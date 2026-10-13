# Native binaries (Mach-O / PE / ELF — C / C++ / Rust / Go)

A native analyst runs `nm`/`strings`/`objdump`, loads the binary in a
decompiler (Ghidra/IDA/Binary Ninja) to recover pseudocode and function
boundaries, and attaches a debugger to observe behavior. Countermeasures, in
order.

## 1. Strip symbols and debug info

Symbols and debug info are the analyst's map. Remove them from release builds.

- **Compile release without debug info** (no `-g`; `strip`-equivalent release
  profile). For Rust: `strip = true`, `debug = false`, `panic = "abort"` in the
  release profile. For Go: `-ldflags="-s -w"` to drop the symbol table and DWARF.
- **Strip** the final binary (`strip`, or linker `-Wl,-s`). Remove local symbols;
  keep only exports you truly need.
- **Hide internal exports**: default-hidden visibility
  (`-fvisibility=hidden`) and export only the public ABI, so internal functions
  are not named for the analyst.
- **Verify:** run `nm -a` and `strings` on your own output; confirm function
  names, internal strings, and paths are gone.

## 2. Encrypt or elide sensitive strings

Plain strings (keys, endpoints, prompts, SQL, messages) are recovered instantly
by `strings`.

- **Encrypt/encode string literals** and decrypt in memory at use, zeroing after.
  Compile-time string-obfuscation libraries do this. Raises cost — the key is in
  the binary, so it is a speed bump, not a vault.
- **Never embed real secrets** (see architecture guide). Obfuscated keys are
  still extractable from memory at runtime.

## 3. Obfuscate code structure

Make the decompiler's output expensive to read.

- **Control-flow flattening, bogus control flow, instruction substitution** via
  an obfuscating compiler pass (e.g. OLLVM-derived toolchains). Defeats clean
  pseudocode recovery; costs size and speed, so apply to the sensitive functions.
- **Inline and merge** hot sensitive routines to blur boundaries.
- **Avoid standard-library fingerprints** where a signature-matcher would
  identify your logic by known library code.
- Expectation: obfuscation slows a human and breaks naive tooling; it does not
  stop a determined reverser. It converts hours into days/weeks.

## 4. Anti-debugging and anti-instrumentation

Detect and refuse analysis environments — proportional to value, and tested so
it does not break legitimate users.

- **Debugger detection**: OS-specific checks (ptrace self-attach on Linux;
  `PT_DENY_ATTACH` / `sysctl` on macOS; `IsDebuggerPresent`/`NtQuery...` on
  Windows). Respond by degrading or exiting, not an obvious hard crash the
  analyst can pinpoint.
- **Timing checks** to catch single-stepping.
- **Anti-hook / anti-inject**: detect injected libraries and Frida-style
  instrumentation.
- All of these are bypassable by a skilled analyst; they filter out casual and
  automated analysis. Do not let them regress accessibility or crash-reporting.

## 5. Integrity / anti-tamper

Make a modified binary fail.

- **Self-checksum** critical code sections at runtime and refuse to run (or
  silently misbehave) if altered — detects patched binaries. Spread and delay
  the check so it is not a single point to NOP.
- **Sign your binaries** (code signing / notarization) so the OS rejects
  tampered or unsigned builds and users get a trust signal.
- **Server attestation** for networked apps: the client proves integrity to your
  server before it gets useful data (see architecture guide). This is the
  tamper defense that actually holds, because the decision is server-side.

## 6. Packing (last, and carefully)

- **Packers/protectors** (commercial protectors, VM-based obfuscation) compress
  and encrypt the binary, unpacking in memory, and can add virtualization that
  is genuinely expensive to reverse. Trade-offs: they trip antivirus heuristics,
  hurt startup and performance, and complicate your own crash analysis. Use a
  reputable one and only for high-value targets.
- Unpacking is a well-practiced skill; packing raises cost, not impossibility.

## 7. Go and Rust specifics

- **Go** binaries are symbol-rich by default and include a lot of runtime
  metadata; `-ldflags="-s -w"` and release stripping are essential, and even then
  tooling can recover structure — keep secrets out.
- **Rust** release with `strip`/`panic=abort` removes names and panic strings
  (which otherwise leak source paths and logic). Watch for format strings that
  embed file paths.

## Verify like an analyst

1. `nm -a`, `strings`, `objdump`/`otool` on your own release binary — symbols and
   sensitive strings should be absent.
2. Load it in a decompiler: how readable is the pseudocode of the function you
   meant to protect?
3. Attach a debugger and inject a hook — do your detections fire and respond
   sanely?
4. Patch a byte in a protected region — does the integrity check catch it?

Report precisely which functions remain legible and which secrets, if any, are
still recoverable from memory.
