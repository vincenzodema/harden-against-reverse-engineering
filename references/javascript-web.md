# JavaScript / TypeScript / web bundles

A web/JS analyst recovers readable sources from shipped bundles (especially via
source maps), reads string literals for keys and endpoints, and observes runtime
behavior in the browser. Close those in order.

## 1. Architecture first

Shipped JS is the *most* readable artifact there is — it is source, lightly
transformed. **Do not rely on obfuscation to protect logic that matters.** Move
anything valuable to the server (see
[architecture-secrets-licensing.md](architecture-secrets-licensing.md)). What
remains client-side should be UI and orchestration you can afford to have read.

## 2. Source-map hygiene — the single most common leak

A production source map hands an analyst your original, commented TypeScript.

- **Do not deploy source maps to production** (`devtool: false` / no
  `sourcemap`). If you need them for error reporting, upload them to your error
  tracker (Sentry, etc.) and **delete them from the public bundle** — never serve
  `.map` files or reference them in a `//# sourceMappingURL` comment.
- **Verify:** fetch your deployed assets and confirm no `.map` is reachable and
  no `sourceMappingURL` points at one.

## 3. Minify and obfuscate what ships

- **Minify** (esbuild/Terser/SWC) as the baseline: mangles names, drops
  comments and dead code. This alone removes most incidental readability.
- **Obfuscate** high-value client code with a dedicated tool
  (e.g. javascript-obfuscator) when minification is not enough: identifier
  renaming, string-array encoding, control-flow flattening, dead-code injection.
  Costs runtime performance and bundle size — apply selectively to the sensitive
  module, not the whole app.
- **Realistic expectation:** obfuscated JS is slower to read, not unreadable.
  Beautifiers and runtime tracing still recover behavior. It buys time.

## 4. Remove embedded secrets and readable constants

- No API keys, tokens, or private endpoints as string literals (an analyst greps
  the bundle). See the secrets section of the architecture guide.
- String-array encoding raises the cost of reading endpoint URLs and feature
  flags, but the strings are reconstructed at runtime — not a secret store.

## 5. Move hot logic to WebAssembly

- Compiling a sensitive routine to **WASM** (from Rust/C/C++/AssemblyScript)
  yields a binary that is meaningfully harder to read than JS — no identifiers,
  decompilers are less mature. Combine with the native techniques in
  [native-binaries.md](native-binaries.md) (symbol stripping, no debug info).
- Not a vault: WASM is still decompilable and observable at the JS boundary.
  Good for raising cost on an algorithm; useless for hiding a key.

## 6. Runtime hardening (use sparingly)

These harass casual inspection in the browser; a skilled analyst bypasses them,
and they can harm UX and accessibility. Apply only to high-value apps.

- **Debugger detection**: timing checks / `debugger`-statement loops that detect
  an open devtools session. Easily patched; mostly deters casual poking.
- **DevTools/console deterrents**: disabling right-click or console does little
  and annoys users — generally not worth it.
- **DOM/behavior obfuscation**: avoid leaking internal API shapes, verbose error
  messages, or feature flags into the DOM or console in production.
- **Integrity**: serve over HTTPS with Subresource Integrity on third-party
  scripts so a tampered dependency is rejected by the browser.

## 7. Node.js / server-side JS you distribute

If you ship a Node CLI or server others run:

- **Bytecode-compile** entry modules (e.g. V8 snapshot / `bytenode`) so the
  shipped files are compiled V8 bytecode rather than plain source. Raises cost;
  still reversible by a determined analyst and tied to Node/V8 versions.
- **Bundle and minify** into a single file to remove module structure and
  comments.
- Apply the same secret rules — a distributed Node tool is a client.

## Verify like an analyst

1. Download your production bundle. Beautify it. Can you read identifiers, find
   endpoint strings, recover logic? If yes, the obfuscation did not land.
2. Search the deployed site for `.map` files and `sourceMappingURL`.
3. Grep for key/token patterns and internal hostnames.
4. Watch network traffic from your own app: is any secret or entitlement
   decision visible or client-side? Move it server-side.

Report what the pass still exposes. Shipped JS is never a safe place for a
secret — the win is that nothing valuable was there to find.
