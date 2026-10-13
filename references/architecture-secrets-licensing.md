# Architecture, secrets, and licensing

These defenses outrank every obfuscator. A reverse-engineering workflow wins by
reading what you shipped; the strongest answer is to not ship the valuable part.

## 1. Move the valuable logic off the client

The only code an analyst cannot decompile is code they never receive.

- **Keep proprietary algorithms, models, and business rules on a server** you
  control. Expose a thin, authenticated API that returns results, not the
  method. A competitor can observe inputs and outputs but cannot copy the engine.
- **Split the app**: a replaceable client (UI, orchestration) plus a protected
  core (the part worth stealing) behind the API. Design the API so that scraping
  it at scale is itself rate-limited, authenticated, and abuse-detected.
- **For ML**: serve inference from your backend. Shipping model weights to the
  client ships the asset. If on-device inference is required, accept that the
  weights are extractable and protect value another way (freshness, data moat).
- **Cost note:** this adds latency, hosting cost, and an offline-use constraint.
  Weigh it against the value of the secret. It is still the best defense.

## 2. Never ship secrets in client artifacts

Anything in a shipped bundle, binary, or package is public. `strings`, a bundle
unpacker, or a network intercept finds it in minutes.

- **No API keys, private keys, signing secrets, or credentials** in client code,
  config, environment variables baked at build time, or assets. Obfuscating a
  key does not protect it — it is still in memory and in the artifact.
- **Use per-user, short-lived tokens** issued by your server after
  authentication, not a shared secret embedded in the app.
- **Third-party keys** (maps, analytics, payments) that must be client-side:
  restrict them by origin/bundle-id/referrer and set tight quotas and alerts in
  the provider console. Treat them as public and limit blast radius.
- **Rotate anything already shipped.** If it was ever in a released artifact,
  assume it is compromised and rotate now.
- **Verify:** grep your own shipped artifact for key patterns and intercept its
  own traffic before release.

## 3. Enforce entitlement server-side

Client-side license checks are the most reliably defeated protection that exists
— a cracker flips one boolean or NOPs one jump.

- **Validate licenses and feature flags on the server**, per request, tied to an
  authenticated account. The client asks "may this user do X?"; the server
  decides. A patched client cannot grant itself access it was never sent.
- **Gate the actual value behind the server**, not just the UI. If the paid
  feature's logic or data lives client-side, unlocking the UI unlocks the
  feature regardless of the check.
- **Signed, expiring entitlements** (e.g. short-lived signed tokens) for
  offline-tolerant apps: the client caches a server-signed grant it cannot forge
  (public-key verify client-side, private key server-only), with an expiry that
  forces periodic re-validation. Accept that a determined attacker can run the
  last valid grant offline until expiry.
- **Offline-first / no-backend apps** cannot have unbreakable licensing — be
  honest with the user. Best effort: hardware-bound signed license files,
  server activation on install, online re-check when connectivity exists, and
  making cracking more annoying than paying. The business model (subscription +
  server features) does more than any check.

## 4. Protect data and assets

- **Encrypt valuable bundled assets** (proprietary datasets, premium content)
  and decrypt in memory only. The decryption key still ships, so this only
  raises cost — pair it with server-delivered keys for high-value content.
- **Stream premium content** from the server with per-session authorization
  rather than bundling it.

## 5. Make copies attributable and revocable

Assume a copy will escape; make it traceable and killable.

- **Watermark** builds and premium content per customer (unique build
  constants, invisible markers, per-tenant strings) so a leaked copy points to
  its source. Keep some watermarks non-obvious so they survive naive stripping.
- **Telemetry** for integrity-check failures, tamper signals, and anomalous
  usage — so you learn a build was cracked and can respond.
- **Revocation**: server-side account/license revocation and key rotation so a
  compromised or cracked instance can be cut off from server features.

## 6. Legal and operational terms

Technical measures buy time; legal terms buy recourse.

- **EULA / license** prohibiting reverse engineering, redistribution, and
  circumvention, to the extent enforceable in your jurisdiction. Note that some
  jurisdictions grant interoperability/security-research exceptions that a
  blanket ban cannot override — have counsel tailor this.
- **Anti-circumvention**: where your work includes a technological protection
  measure, applicable law (e.g. DMCA §1201 in the US, the EU Software Directive)
  may provide additional recourse against those who defeat it.
- **Takedown and enforcement** process ready in advance for leaked copies and
  clones.

## Priority recap

1. Move secrets and valuable logic server-side (defeats all client analysis).
2. Ship no embedded secrets; rotate what already shipped.
3. Enforce entitlement on the server.
4. Encrypt/stream valuable assets.
5. Watermark, instrument, revoke.
6. Legal terms and a takedown process.

Only after these do client-side obfuscation and anti-tamper (the platform
guides) earn their keep — as cost multipliers on what you are forced to ship,
never as the primary line of defense.
