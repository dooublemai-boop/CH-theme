# Codexhost Web Theme Proposal

This repository contains the approved-but-unimplemented design baseline and implementation plan for an external visual theme extension for Codexhost Web/PWA.

- [Design baseline](docs/proposals/web-theme-extension-design.md)
- [Implementation plan](docs/proposals/web-theme-extension-implementation-plan.md)

These documents are planning material, not an implementation. To execute the plan, open the Codexhost source repository on the target computer and first re-check the paths, APIs, build wiring, tests, and runtime versions against that checkout. The plan has implementation gates for the public browser-safe contract package, shared client module resolution, runtime-version snapshots, the semantic token allowlist, and the first-paint bootstrap path. Resolve those gates before coding.

The intended scope is Codexhost's own Web/PWA UI across Harness sessions. It does not theme Codex Desktop's native UI or a Harness's own CLI, TUI, or desktop interface, and it does not change message content or execution semantics.
