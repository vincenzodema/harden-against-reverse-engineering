# .NET / managed assemblies (PE/CLI)

Managed assemblies are the easiest native-looking target to reverse: a decompiler
(ILSpy/dnSpy/dotPeek) reconstructs near-original C#/VB from the IL and rich
metadata. Countermeasures, in order.

## 1. Obfuscate the IL and metadata
- **Rename** types, methods, fields, and parameters to meaningless or
  unprintable names (an obfuscator such as a ConfuserEx-class or commercial
  tool). This removes the single biggest readability advantage of managed code.
- **Control-flow obfuscation**: rewrite method bodies so the decompiler emits
  tangled or invalid-looking C#.
- **String encryption**: encrypt literal strings, decrypt at runtime — keeps
  endpoints, SQL, and messages out of a plain metadata read.
- **Anti-decompile / anti-debug**: options that break common decompilers and
  detect dnSpy-style debugging. Bypassable; raises cost.
- **Resource/assembly encryption and merging** to hide auxiliary assemblies.

Expectation: obfuscation turns a clean decompile into expensive, error-prone
reading. Deobfuscators exist for popular tools — it buys time, not immunity.

## 2. Compile ahead-of-time to native
- **NativeAOT** (or an IL-to-native path) ships real native code instead of IL,
  removing most managed metadata and forcing the analyst onto a native
  decompiler — a large step up in cost. Then apply
  [native-binaries.md](native-binaries.md): strip, string-encrypt, integrity.
- Trade-offs: build/runtime constraints, larger binaries, reflection limits.
  Worth it for the component that actually holds value.

## 3. Secrets and entitlement
- **No keys, connection strings, or secrets** in the assembly, `app.config`, or
  resources — a metadata reader finds them instantly, obfuscated or not. Rotate
  anything shipped. Keep secrets server-side.
- **License checks server-side.** A client-side `IsLicensed` is trivially
  patched in IL; the paid logic or data must live behind an authenticated server
  call (see [architecture-secrets-licensing.md](architecture-secrets-licensing.md)).

## 4. Integrity
- **Strong-name + Authenticode signing** so tampered assemblies are detectable
  and the OS/user gets a trust signal. Strong naming alone is removable by a
  determined attacker — pair with signing and server attestation.
- **Runtime self-integrity** checks on critical assemblies, with a non-obvious
  failure response.

## Verify like an analyst
1. Open your own release assembly in a decompiler: are names meaningless,
   strings encrypted, logic tangled? If it reads like your source, obfuscation
   did not apply to the release configuration.
2. `strings` and a metadata dump: no secrets, no connection strings.
3. Patch the license check in IL and run: does the paid feature still unlock? If
   yes, the entitlement is client-side — move it to the server.
