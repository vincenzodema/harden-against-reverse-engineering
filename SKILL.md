---
name: harden-against-reverse-engineering
description: Harden your own software so it is costly to copy, clone, or reverse engineer. Use when the user wants to protect an app, tool, binary, bundle, or package they own or are authorized to ship — obfuscation, symbol stripping, anti-tamper/integrity checks, secret removal, licensing, and server-side hardening. The mirror image of a reverse-engineering workflow: it closes the vectors that decompilers, bundle unpackers, and runtime observers exploit. Not for defeating someone else's protections or analyzing software you do not own.
metadata:
  version: "1"
---

# Harden Against Reverse Engineering

Use this skill to make software **you own or are authorized to ship** expensive
to reverse engineer, clone, or tamper with. It is the defensive mirror of a
reverse-engineering workflow: for each vector an analyst uses to read a shipped
artifact — decompilation, bundle unpacking, symbol and string recovery, runtime
observation, network capture — it prescribes a concrete countermeasure.

## Scope and authorization

Apply these techniques only to artifacts the user built or is authorized to
harden. This skill protects intellectual property and raises the cost of
cloning and tampering. It is **not** for stripping another vendor's protections,
hiding malware from defenders, evading security analysis of software shipped to
others, or circumventing DRM you do not own. If the request is to obscure
software from the people who run it or from security tooling, stop and ask what
is being protected and from whom.

## The honest threat model — read this first

**You cannot make client-shipped code impossible to reverse engineer.** Anything
that runs on a machine the attacker controls can, given enough time, be read.
Obfuscation, packing, and anti-debugging raise *cost and time*; they do not
create *impossibility*. Any skill, tool, or vendor promising "uncrackable" or
"impossible to reverse" is selling a false model. Set expectations honestly with
the user.

What actually works, in order of effectiveness:

1. **Don't ship the secret.** Code and data that never reach the client cannot
   be extracted. Move proprietary logic, models, keys, and valuable algorithms
   to a server you control; expose only a thin, authenticated API. This is the
   single highest-leverage defense and defeats *every* client-side analysis
   vector at once.
2. **Make what you must ship costly to read.** Minify, obfuscate, strip symbols,
   encrypt strings and assets. Converts "an afternoon" into "weeks."
3. **Detect and respond to tampering.** Integrity checks, anti-debugging, and
   runtime attestation so a modified or instrumented build fails or phones home.
4. **Enforce entitlement server-side.** Licensing and feature gating validated by
   a server, never by a client-side boolean a cracker can flip.
5. **Make cloning legally and operationally costly.** Watermarking, telemetry,
   EULA/anti-circumvention terms, and the ability to revoke.

Defense in depth: assume each client-side layer will eventually fall, and make
sure the thing that actually matters (secrets, entitlement) does not live there.

## Route the target first

Pick the guide that matches what the user ships. Match the same artifact types a
reverse-engineering workflow routes on:

- **Web app / JavaScript / TypeScript bundle (browser or Node):**
  [references/javascript-web.md](references/javascript-web.md) — minification,
  obfuscation, source-map hygiene, moving logic to WASM or server, runtime
  anti-debugging.
- **Electron / desktop JS (ASAR, packaged tree):**
  [references/electron-desktop-js.md](references/electron-desktop-js.md) — ASAR
  integrity, bytecode compilation, main-process secrets, fuses.
- **Native binary (Mach-O / PE / ELF, C/C++/Rust/Go):**
  [references/native-binaries.md](references/native-binaries.md) — symbol
  stripping, control-flow obfuscation, string encryption, anti-debug,
  integrity/anti-tamper, packers.
- **Mobile app (Android APK / iOS IPA):**
  [references/mobile-apps.md](references/mobile-apps.md) — R8/ProGuard, native
  checks, root/jailbreak and hook detection, RASP, App/Play integrity.
- **.NET / managed assembly (PE/CLI):**
  [references/managed-dotnet.md](references/managed-dotnet.md) — IL obfuscation,
  renaming, anti-decompile, NativeAOT.
- **Packaging, assets, licensing, and secrets (all targets):**
  [references/architecture-secrets-licensing.md](references/architecture-secrets-licensing.md)
  — the architecture-level defenses that outrank any obfuscator.

If the user says only "my app," ask for the platform and how it is distributed
(web, downloadable binary, mobile store, desktop installer) before prescribing
anything — the correct countermeasures differ entirely by delivery channel.

## Verify the way an attacker would

A countermeasure is only real if it survives the matching attack. After each
change, verify by **doing to your own build what an analyst would do**:

- Minified/obfuscated JS → unpack the bundle, look for readable identifiers,
  check that no source map ships to production and that string literals for keys
  or endpoints are gone.
- Stripped native binary → run `nm`, `strings`, and a decompiler against your
  own output; confirm symbols and sensitive strings are absent and that function
  boundaries are hard to recover.
- Mobile build → decompile your own release artifact; confirm renaming applied,
  no debug flags, and detections fire under a debugger/emulator/hook.
- Secrets → grep the shipped artifact and intercept its own network traffic;
  confirm no long-lived key, no hardcoded endpoint auth, no entitlement decided
  client-side.

If your own cheap pass recovers the thing you meant to protect, the measure
failed. Report honestly what the pass still exposes rather than claiming the
artifact is "protected."

## Work in order of leverage

Do not start with a fancy obfuscator. Start with the architecture:

1. **Inventory what you ship and what must stay secret.** List the artifacts that
   reach the client and the assets inside them (keys, endpoints, algorithms,
   models, business rules). For each secret, ask: *can this move server-side?*
   If yes, that beats any obfuscation.
2. **Remove secrets that should never ship.** Rotate anything already shipped —
   assume it is already public.
3. **Enforce entitlement and licensing server-side.** Never gate paid features
   on a client-side check alone.
4. **Harden the client artifact** with the platform guide: strip, obfuscate,
   encrypt strings/assets.
5. **Add tamper detection and response** proportional to the value at risk.
6. **Watermark and instrument** so copies are attributable and abuse is visible.
7. **Set legal and operational terms** (EULA, anti-circumvention, revocation).

Match effort to value: a free internal tool needs step 1–2; a paid desktop app
with a valuable algorithm needs the whole chain, with the algorithm server-side.

## Finish the task

State plainly, for each protected asset, *which vector it now resists and which
it does not*. Distinguish "moved off the client (cannot be extracted)" from
"made costly to read (will eventually fall)." Never tell the user something is
"uncrackable" or "impossible to reverse engineer" — give them the real cost
curve and where the remaining exposure is. Recommend re-verifying after every
dependency or build-tool upgrade, since those routinely reintroduce symbols,
source maps, and debug flags.
