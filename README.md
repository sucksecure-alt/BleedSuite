# Bleed Suite

A native intercepting proxy for macOS — the practical subset of Burp Suite, written
in Swift with no runtime dependencies.

An explicit HTTP/HTTPS proxy that sits between your browser and the network. It
terminates TLS locally with its own certificate authority, records every exchange,
lets you hold and rewrite traffic in flight, and resend requests with parameters
changed.

```
open dist/Bleed Suite-1.0.0.dmg     # drag to Applications, launch
```

---

## What it does

| Workspace | Purpose |
|---|---|
| **Dashboard** | Proxy status, CA trust, traffic counters, event log |
| **Intercept** | Hold requests and responses, edit them, forward or drop |
| **HTTP history** | Every transaction, filterable, with a full inspector |
| **Target** | Auto-built site map of every host and endpoint seen |
| **Repeater** | Manual request editor, tabs, response history |
| **Comparer** | Field-level diff between any two exchanges |
| **Intruder** | Payload positions `§`, dictionaries, threads, match rules |
| **Sequencer** | Generate payload sequences for the Intruder |
| **Scanner** | 12 passive checks on captured traffic, 4 active probes |
| **Match / Replace** | Rewrite any part of any message, with a test area |
| **Settings** | Port, CA management, system proxy, scope, export |

Export: HAR 1.2, Burp-style CSV, sitemap XML, scanner issue XML, project files.
Import: HAR and CSV back into the history.

---

## Build from source

Requires macOS 14+ and the Swift 6 toolchain (Xcode 16 or Command Line Tools).

```bash
git clone <repo> && cd BleedSuite3

./build.sh          # debug build with errors only
./package.sh        # release build → dist/Bleed Suite.app + .dmg
./package.sh --test # run the engine self test
```

`package.sh` produces a universal binary (arm64 + x86_64), ad-hoc signs it with
non-sandbox entitlements, and builds a styled DMG.

Prebuilt DMGs are attached to tagged releases and archived by CI, so end users do
not need a toolchain at all.

> **Note on the SDK.** Command Line Tools installs ship no `SwiftUIMacros` plugin,
> which every `@State` in SwiftUI needs. Both scripts therefore prefer the
> `MacOSX26.x` SDK when one is present. With full Xcode installed this is automatic.

---

## How the engine works

**TLS interception.** Network.framework cannot upgrade an already-accepted socket to
TLS, so the client's encrypted stream is spliced into one end of a `socketpair`
while Secure Transport runs the TLS server handshake on the other
(`Sources/Core/TLSTermination.swift`). The decrypted stream re-enters the exact
same HTTP pipeline as plaintext traffic.

**Certificate authority.** The Root CA and every leaf are produced by the Security
framework alone — no `openssl` binary, no temporary key material on disk
(`Sources/Core/CertAuthority.swift`). Leaf keys live in a dedicated keychain file
inside Application Support rather than the login keychain, and the cache is
bounded with eviction, so a long session does not accumulate hundreds of permanent
key items.

**Connection handling.** Client connections are served with keep-alive and HTTP
pipelining; upstream connections come from a bounded idle pool, so browsing does not
pay a TCP + TLS handshake per request. Every connect has a deadline, so a
black-holed origin yields a 502 instead of a hang.

**Body integrity.** Bodies are always `Data`. Binary payloads survive a round trip
through the editors untouched, and chunked responses are decoded and re-encoded
correctly rather than truncated on the wire.

---

## Layout

```
Sources/
  Core/          engine — no UI imports
    HTTPParsing     HTTP/1.1 parser, chunked codec, framing, hop-by-hop rules
    ProxyServer     listener, per-connection sessions, MITM, upstream pool
    ByteStream      NWConnection as an async stream
    TLSTermination  Secure Transport server over a socketpair
    CertAuthority   Root CA, leaf issuance, trust management
    RulesEngine     Match and Replace, plus the scope filter
    InterceptQueue  hold / edit / release
    HistoryStore    thread-safe history with an indexed filter
    BodyStore       SQLite spill for large bodies, project files
    Scanner         passive and active checks
    Intruder        payload positions, dictionaries, sweeps
    Spider          link discovery and crawling
    Exporter        HAR, CSV, XML, project JSON
    System          networksetup / security wrappers
    Repeater        manual request engine, comparer
    Models          HTTP message model, site map, issues, rules
    SelfTest        129 engine checks, run with --selftest
  UI/            native front end (SwiftUI + AppKit)
    Theme           design tokens and the shared colour ramp
    AppModel        application state and the event sink
    RootView        sidebar, workspaces, site map
    …               one file per workspace, AppKit where the row counts are large
```

---

## Testing

```bash
./package.sh --test
```

129 checks covering wire-format edge cases (chunked with trailers, pipelining,
binary bodies, 204/304, 101 upgrades, obs-fold), storage and filtering, rule
evaluation, CA issuance and chain verification, editor round trips, the interception
queue, and a live proxy suite that boots a real listener and drives real exchanges
through it — including keep-alive reuse and rule rewriting observed at the origin.

---

## Known limitations

- **HTTP/2 is not spoken.** ALPN advertises `http/1.1` only, so h2-only origins are
  downgraded. Sites still work; large responses are slower.
- **No HTTP/3.** QUIC bypasses the proxy entirely.
- **WebSocket frames are tunnelled, not decoded** — the upgrade succeeds and traffic
  flows, but frames are not shown or editable.
- **Passive-only DOM analysis.** XSS probing covers reflected, not stored or DOM
  sinks.
- **No extension API.** Burp extensions in Java/JRuby are not supported.

---

## Legal

For testing systems you own or have permission to test. Intercepting traffic
without authorisation may be illegal in your jurisdiction.