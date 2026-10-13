# harden-against-reverse-engineering

A Claude Code / Claude Agent **skill** for hardening software **you own or are
authorized to ship** against copying, cloning, and reverse engineering.

It is the defensive mirror of a reverse-engineering workflow: for every vector
an analyst uses to read a shipped artifact — decompilation, bundle unpacking,
symbol and string recovery, runtime observation, network capture — the skill
prescribes a concrete countermeasure, routed by platform.

## What it covers

- **Web / JavaScript / TypeScript bundles** — source-map hygiene, minification,
  obfuscation, WASM, runtime hardening.
- **Electron / desktop JS** — ASAR integrity, fuses, bytecode, renderer
  hardening.
- **Native binaries (Mach-O / PE / ELF)** — symbol stripping, control-flow
  obfuscation, string encryption, anti-debug, integrity/anti-tamper, packing.
- **Mobile (Android / iOS)** — R8/ProGuard, native checks, RASP, certificate
  pinning, Play Integrity / App Attest.
- **.NET / managed assemblies** — IL obfuscation, renaming, NativeAOT.
- **Architecture, secrets, and licensing** — the server-side defenses that
  outrank any obfuscator.

## The honest premise

You **cannot** make client-shipped code impossible to reverse engineer. Anything
that runs on a machine the attacker controls can eventually be read. This skill
raises *cost and time* and, most importantly, keeps the things that actually
matter — secrets and entitlement — off the client entirely. Any tool promising
"uncrackable" is selling a false model.

## Scope

For protecting intellectual property and raising the cost of cloning and
tampering on software you own or are authorized to harden. **Not** for stripping
another vendor's protections, hiding software from the people who run it or from
security tooling, or analyzing software you do not own.

## Install

Clone into your Claude skills directory (or your agent's skill path):

```bash
git clone https://github.com/<you>/harden-against-reverse-engineering.git \
  ~/.claude/skills/harden-against-reverse-engineering
```

The skill activates when you ask Claude to protect, harden, or anti-reverse an
app, binary, bundle, or package you own.

## Structure

```
SKILL.md                                    # routing + method + honest threat model
references/
  architecture-secrets-licensing.md         # highest-leverage defenses
  javascript-web.md
  electron-desktop-js.md
  native-binaries.md
  mobile-apps.md
  managed-dotnet.md
```

## License

MIT — see [LICENSE](LICENSE).
