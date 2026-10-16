# Contributing

## Before you open a pull request

```bash
./package.sh --test     # must print PASS
./build.sh             # must produce no errors
```

The self test is the contract: 129 engine checks, including a live proxy suite that
binds a real port and drives real exchanges. New engine behaviour needs a check in
`Sources/SelfTest.swift`.

## Style

Match the file you are editing. Broadly:

- Comments explain *why*, not *what*. The code already says what it does.
- No `fatalError` in production paths — degrade and report instead.
- Prefer `Data` over `String` for anything that came off the wire; binary bodies
  must never be routed through `String`.
- Colours come from `Theme` / `Palette`. Do not introduce new hex literals in views;
  the AppKit-backed components cannot convert a SwiftUI `Color` outside a render pass.

## Layout rules

- `Sources/Core` must not import SwiftUI. It is the engine and is testable headlessly.
- `Sources/UI` holds one file per workspace. If a view needs an AppKit component,
  the reason belongs in a comment — `List` is unusable at 60 000 rows.
