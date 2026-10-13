# Mobile apps (Android APK / iOS IPA)

A mobile analyst unzips the package, decompiles Dalvik/ART bytecode (Android) or
disassembles the Mach-O and reads class metadata (iOS), reads resources and
strings, and uses dynamic instrumentation (Frida) and emulators/rooted devices.
Store-side integrity APIs are your strongest client-side allies.

## Android

### 1. Shrink, rename, obfuscate
- **Enable R8** (minify + resource shrinking) in the release build:
  `minifyEnabled true`, `shrinkResources true`, with a tuned `proguard-rules`.
  This renames classes/methods/fields and strips unused code — the baseline.
- **Keep rules tight**: only `-keep` what reflection/serialization truly needs;
  every needless keep leaves a readable name for the analyst.
- For higher value, a **commercial hardener** (DexGuard-class) adds string
  encryption, class encryption, control-flow obfuscation, and reflection hiding
  beyond what R8 does.
- **Native code**: move sensitive logic into an NDK `.so` and apply the
  [native-binaries.md](native-binaries.md) techniques — harder to reverse than
  dex, though still reversible.

### 2. Strings, resources, secrets
- Strip or encrypt sensitive strings; `strings` on the dex/so and a resource
  decoder read plaintext instantly.
- **No secrets in the APK** — not in code, `res/`, `assets/`,
  `BuildConfig`, or the manifest. Restrict any unavoidable third-party key by
  package/signature in the provider console. Rotate shipped keys.

### 3. Runtime self-protection (RASP)
- **Root detection**, **emulator detection**, **debugger detection**, and
  **Frida/hook detection**, responding by degrading or refusing sensitive flows.
  All bypassable; they raise cost and filter automated abuse.
- **Repackaging/signature check**: verify the signing certificate at runtime so
  a resigned clone is detected.
- **Certificate pinning** so traffic cannot be trivially intercepted and the API
  cannot be cloned from a proxy capture.

### 4. Platform integrity (the strong one)
- **Play Integrity API**: let your server verify the app binary, the device, and
  the account are genuine before serving valuable responses. Because the
  verdict is checked server-side, a patched client cannot fake it. This is the
  Android tamper defense that actually holds.

## iOS

### 1. Build hygiene
- **Release build**: strip debug symbols, `DEPLOYMENT_POSTPROCESSING`/`STRIP`
  settings on; don't ship dSYMs inside the IPA (upload to your crash service).
- Objective-C class/method names are rich metadata an analyst reads with
  class-dump-style tools — **Swift** gives somewhat less introspectable metadata;
  prefer it for sensitive logic and strip/mangle where possible.
- Move hot logic to **C/C++/Rust** compiled in and apply the native guide.

### 2. Strings and secrets
- Encrypt sensitive strings; `strings` on the decrypted Mach-O finds plaintext.
- **No secrets in the IPA or `Info.plist`.** Note App Store encryption only
  protects the on-disk binary from casual copying — on a jailbroken device a
  decrypted dump is available to the analyst. Keep secrets off the device.

### 3. Runtime self-protection
- **Jailbreak detection**, **debugger detection** (`ptrace PT_DENY_ATTACH`,
  `sysctl`), and **Frida/hook detection**, with graceful response.
- **Certificate pinning** against traffic interception and API cloning.

### 4. Platform integrity (the strong one)
- **App Attest / DeviceCheck**: have your server verify a hardware-backed
  assertion that the genuine app on a genuine device is calling, before serving
  valuable data. Server-side verdict — the defense that holds.

## Cross-cutting
- The store package is a ZIP: assume everything inside is readable. The real
  protection for paid features and secrets is **server-side entitlement +
  platform integrity** (see
  [architecture-secrets-licensing.md](architecture-secrets-licensing.md)).
- RASP and pinning raise the cost of cloning and abuse; they do not make the app
  uncrackable, and they must be tested on real devices so they never lock out
  legitimate users.

## Verify like an analyst
1. Decompile your own release APK (jadx-class tooling) / disassemble your IPA:
   are names renamed, strings encrypted, secrets absent?
2. Run on a rooted/jailbroken device and under Frida: do detections fire and
   respond without an obvious single bypass point?
3. Proxy the traffic: does pinning hold, and is any entitlement decided
   client-side?
4. Resign/repackage a clone: does the signature check catch it?
