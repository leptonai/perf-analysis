# Security Policy: leptonai/perf-analysis

## Reporting a Vulnerability

Do not open a public GitHub issue or pull request for a suspected vulnerability, in this repository or in the upstream issue tracker that `README.md` and `CONTRIBUTING.md` link to.

Report vulnerabilities privately through one of these channels:

1. **NVIDIA Vulnerability Disclosure Program (preferred):** [NVIDIA Product Security](https://www.nvidia.com/en-us/security/), using the [vulnerability report form](https://www.nvidia.com/en-us/security/report-vulnerability/).
2. **Email:** [psirt@nvidia.com](mailto:psirt@nvidia.com). Use the [NVIDIA public PGP key](https://www.nvidia.com/en-us/security/pgp-key) when the report contains sensitive evidence.
3. **GitHub private vulnerability reporting:** This repository's **Security** tab, then **Report a vulnerability**.

Include the repository name, the affected branch and commit, the vulnerability type, reproduction steps, proof of concept if available, and the impact. Remove VM service URIs, tokens, and trace files from real applications from examples. Use the private channel to share anything that is itself sensitive.

NVIDIA PSIRT acknowledges receipt, validates the report, and coordinates remediation and disclosure. For this repository, remediation is a source change on the affected branch. If the issue is in code inherited unchanged from upstream Dart & Flutter DevTools, say so in the report; the Flutter project describes its own process at <https://docs.flutter.dev/security>.

## Security Architecture and Context

This repository is a fork of [flutter/devtools](https://github.com/flutter/devtools), the Dart & Flutter DevTools suite of debugging and performance tools. It is an application: a Flutter web app with supporting Dart packages. This fork publishes no release and no package of its own.

There are two branches:

- `master`, the default branch, is an unmodified snapshot of upstream from February 2025.
- `perf-analysis` adds three commits on top of `master`. All of them edit the precompiled Perfetto trace viewer under `third_party/packages/perfetto_ui_compiled/lib/dist/`: the dark theme styles, the removal of a call that looks for attached Android devices, and the list of web origins the viewer trusts.

The main parts inherited from upstream are:

- `packages/devtools_app`, the DevTools application. It connects to the Dart VM Service of a running app. Its performance screen embeds the Perfetto viewer (`lib/src/screens/performance/panes/timeline_events/perfetto/`).
- `packages/devtools_shared`, `packages/devtools_app_shared`, `packages/devtools_extensions`, and `packages/devtools_test`, the shared libraries, the extension framework, and the test support.
- `third_party/packages/perfetto_ui_compiled`, the compiled Perfetto UI (version 34.0) and its DevTools theme files.
- `tool/` and `.github/workflows/`, the upstream build, release, and automation scripts. GitHub Actions is not enabled in this fork at the time of writing.

Its security responsibility has two parts. DevTools holds full debugging authority over the app it connects to. The Perfetto viewer parses trace data handed to it by the page that embeds it.

The boundaries are:

- The connection from DevTools to a Dart VM Service, addressed by a URI that carries its own secret.
- The `postMessage` channel between the Perfetto viewer and the page that embeds it, filtered by the origin check `isTrustedOrigin` in the compiled `frontend_bundle.js`.
- The frames in which DevTools loads extensions.
- The trace files and snapshots a user opens.

**Repository Exposure Classification:** Public. Basis: `github.com/leptonai/perf-analysis` is a public GitHub repository; this document is written to public-safe detail.

**Service Exposure Classification:** External / Regulated (medium confidence). Basis: public fork of an externally distributed developer tool; the fork has no release or package of its own, and its `perf-analysis` branch changes which web origins the bundled trace viewer accepts data from. The source does not show where a build of that branch is hosted.

### Threat Model

1. **The trace viewer accepts data from more origins on `perf-analysis`:** `isTrustedOrigin` in `third_party/packages/perfetto_ui_compiled/lib/dist/v34.0-16f63abe3/frontend_bundle.js` decides which pages may send a trace over `postMessage` without the user confirming. The branch replaces the upstream list with Lepton-operated origins, and one entry is a suffix match that covers every subdomain of a domain. Any page served from a matching origin can load a trace into the viewer unprompted, which makes a parsing defect in the viewer reachable from all of those pages.
2. **The compiled bundle is edited by hand:** The branch changes the generated `frontend_bundle.js` directly instead of rebuilding Perfetto from source, and the first commit also changes whitespace across the file. A reviewer cannot confirm from this repository that the bundle equals the upstream build plus the three stated changes.
3. **The snapshot trails upstream:** `master` is several hundred commits behind `flutter/devtools`, and the bundled Perfetto UI is fixed at version 34.0. Security fixes made upstream after February 2025 are absent until the fork is updated.
4. **DevTools has debugging authority over the connected app:** A DevTools session attached to a VM Service can evaluate code in the app, read its memory, and inspect its network traffic. The VM Service URI works as a bearer secret. Sharing a session, a URI, or a saved snapshot gives the recipient that access or that data. This behavior is inherited from upstream.
5. **Inherited workflows assume the upstream repository:** The files in `.github/workflows/` were written for `flutter/devtools` and its automation accounts. Several run on a schedule, push commits, or open pull requests. Enabling GitHub Actions on this fork would run them with this fork's token and secrets.
6. **Inherited documents send reports to the wrong place:** `README.md`, `CONTRIBUTING.md`, and `CODEOWNERS` point to the upstream issue tracker and reviewers. Someone who follows them would disclose a problem in this fork's changes in a public upstream issue.

### Critical Security Assumptions

- The upstream code on `master` is trusted as published. This fork does not audit it.
- A developer runs DevTools against their own app on a machine and network they trust, and keeps the VM Service URI private.
- Every origin matched by the `perf-analysis` origin check is controlled by the operator and serves no untrusted content, user uploads included.
- Trace files may be untrusted. The safety of parsing them is inherited from Perfetto UI 34.0.
- GitHub Actions stays disabled on this fork, or the workflows are reviewed before it is enabled.
- A hosted build of the `perf-analysis` branch is served over HTTPS by the operator. Nothing in this repository configures that hosting.

## Supported Versions

This fork has no tags and no releases of its own. `master` follows an upstream snapshot and `perf-analysis` carries the fork's changes. Security reports should name the branch and commit. Fixes for code inherited from upstream arrive when the fork is updated from upstream.
