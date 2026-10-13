# Electron and packaged desktop JavaScript

An Electron analyst extracts the `app.asar`, reads the JS inside (it is usually
plain source), inspects the renderer like any web page, and attaches to the main
process. Electron ships a full browser plus your source, so treat it as the most
exposed desktop target and lean hard on architecture.

## 1. Architecture first
- Electron hands the user your source in a ZIP-like archive. **Nothing valuable
  should live in the renderer.** Keep secrets and paid logic server-side (see
  [architecture-secrets-licensing.md](architecture-secrets-licensing.md)); the
  app is a client.

## 2. Pack and compile the JS
- **Bundle and minify** main and renderer code (esbuild/webpack) so the archive
  holds one mangled file per process instead of your readable module tree.
- **Apply the web guide** to the renderer:
  [javascript-web.md](javascript-web.md) — no production source maps, string
  handling, obfuscation of sensitive modules.
- **Bytecode-compile** the main-process entry (V8 snapshot / `bytenode`) so the
  shipped logic is compiled bytecode, not source. Raises cost; still reversible
  and version-tied.
- **Move hot logic to a native addon** (N-API `.node`) or WASM and apply
  [native-binaries.md](native-binaries.md).

## 3. Protect the ASAR
- **ASAR integrity**: enable Electron's ASAR integrity feature so a swapped or
  edited `app.asar` is rejected at launch (pair with code signing — unsigned
  tampering then fails the OS check too).
- Remember ASAR is an archive, **not encryption** — packing into ASAR hides
  nothing by itself. Integrity detects tampering; it does not stop reading.

## 4. Electron fuses and hardening
- **Flip Electron fuses** to disable attack surfaces in release:
  turn off `RunAsNode`, `EnableNodeCliInspectArguments`, and
  `EnableNodeOptionsEnvironmentVariable` so the packaged app cannot be relaunched
  as a general Node/inspector shell; enable `OnlyLoadAppFromAsar` and cookie
  encryption.
- **Disable DevTools** in production builds and do not ship an inspector flag.
- **Secure the renderer**: `contextIsolation: true`, `nodeIntegration: false`,
  a strict `preload` exposing only a minimal API, and a tight Content Security
  Policy — this limits what a tampered or injected renderer can reach.

## 5. Code signing and integrity
- **Sign and notarize** the app (macOS) / Authenticode-sign (Windows) so a
  modified build is rejected and users get a trust signal.
- For networked apps, **server attestation** before serving valuable data is the
  tamper defense that holds.

## Verify like an analyst
1. `npx asar extract` your own `app.asar`: is the JS readable? Are there source
   maps or secrets inside?
2. Try launching the packaged binary as Node / with an inspector flag — do the
   fuses block it?
3. Edit a file and repack the ASAR — does integrity + signing reject it at
   launch?
4. Open the renderer's devtools (if reachable) and inspect the exposed preload
   API surface and network traffic for secrets or client-side entitlement.
