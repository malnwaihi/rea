---
name: rea-investigator
description: Disciplined workflows on top of REA (reverse-engineer-anything) for root-cause debugging of shipped software, security review of binaries/packages/firmware, research-backed feature reconstruction, and UI-motion extraction. Use with REA when the question is "why does this shipped app/binary do X", "is this binary/package/firmware safe", "how does feature X work and build it", or when /investigate, /gstack cso, /deep-research, /emil-design-eng or /fortinet-security-expert need evidence from an artifact without source.
---

# REA Investigator

A companion to REA's own `reverse-engineer-anything` skill. That skill owns tool routing, connection recovery and Evidence mechanics — read it first and follow it. This skill adds the *method*: how to turn REA output into a defensible answer, a verified fix, or a security finding.

It lives as a separate skill on purpose: `rea setup` / `rea doctor` replace the shipped `reverse-engineer-anything` copy, so edits made there are overwritten. Nothing here changes REA's tool contracts.

## Authorization gate (always first)

Before opening any target, confirm it is software the user owns, builds, or is authorized to examine (own product, licensed vendor appliance under a support/assessment agreement, open-source build, CTF). If ownership or authorization is unclear, ask once. Do not help bypass licensing, DRM, activation or anti-tamper, or extract credentials/keys from third-party products. Commercial engines (IDA, Hopper, JEB) need valid licenses.

## Pick the mode

| Request | Mode | Borrowed discipline |
|---|---|---|
| Shipped app/binary misbehaves, crash, regression between versions | **debug** | /investigate iron law |
| "Is this binary / package / plugin / firmware safe", supply-chain check | **audit** | /gstack cso confidence gate |
| "How does feature X work", prior art, compare products | **research** | /deep-research triangulation |
| Copy a feel: animation, gesture, transition, micro-interaction | **motion** | /emil-design-eng |
| Network appliance image, FortiGate/FortiOS artifacts, embedded firmware | **firmware** | /fortinet-security-expert |

Run only the matching section. Every mode shares the ledger and report below.

## Shared rules

1. **Hypothesis before drilling.** Write one testable claim ("the crash comes from X dereferencing Y when Z") before deep tool calls. Each REA call should confirm or kill a named hypothesis.
2. **Three-strike rule.** Three refuted hypotheses → stop, show the ledger, and ask whether to widen scope, switch to runtime observation, or accept a bounded unknown.
3. **Evidence or it didn't happen.** Every conclusion cites REA Evidence IDs (or `parent_evidence_id` + view). A claimed limitation ("REA can't read this") needs the verbatim REA result or `binary_session` availability reason.
4. **Static ≠ runtime.** Never present static analysis as observed execution. Escalate to runtime tools only when static evidence cannot answer, and only within the declared target and actions.
5. **External sources are labeled.** Docs, advisories, changelogs and forum posts are context, never Evidence. Mark them `[ext]` with a link.
6. **Don't repeat unchanged calls.** Change an input or resolve a blocker first (REA's own rule).

### Finding ledger

Keep one table during the work and include it in the report:

| # | Claim | Type (observed / inferred / unknown) | Evidence IDs or [ext] link | Confidence 1–10 | Next probe |
|---|---|---|---|---|---|

Confidence: 9–10 = quoted from returned pseudocode/bytes/records; 7–8 = consistent across two independent Evidence types; 5–6 = single inference, flag "verify"; ≤4 = keep out of conclusions.

## Mode: debug

1. **Symptom.** Exact error, crash report, log line, version/build hash. For `.ips` crashes use `inspect_apple_ips`; for Linux cores `inspect_recorded_crash`.
2. **Locate.** `open_binary` → `search_strings` / `xrefs` from the error text or symbol → `analyze_function` on the owner. For JS/Electron, `analyze_javascript_application` (summary) → `trace_application_feature` from the seed.
3. **Regression?** If a previous version worked, diff it: `compare_application_versions` (JS) or `compare_functions` on the matched procedures. A regression's cause is in the diff.
4. **Hypothesis test.** Confirm with a narrower view or a runtime capture of the exact path. If refuted, back to step 2 (three-strike rule).
5. **Fix scope.** If the user owns the source, fix it there with a regression test that fails before and passes after. If not, deliver a workaround (config, input, version pin) plus a vendor-ready bug report with Evidence references — never a binary patch of third-party software.

## Mode: audit

1. **Mental model.** Identity first: hashes, signer (`inspect_pe_certificates`), imports/exports (`inspect_pe_imports`, `inspect_native_exports`), embedded dependency metadata (`inspect_rust_binary`, `inspect_go_binary`), Electron fuses (`inspect_electron_fuses`), APK signing (`inspect_apk_signature`).
2. **Attack-surface census.** Network listeners/URLs in strings, IPC handlers and preload bridges (JS), native add-ons, update channels, privileged helpers, shell/process spawns, file-write paths.
3. **Category pass.** Command/shell injection sinks, unsafe deserialization, hard-coded secrets or endpoints, disabled mitigations (`inspect_pe_load_config`, `inspect_binary_layout`), outdated bundled libraries (match versions to public advisories `[ext]`), Electron misconfig (nodeIntegration, contextIsolation, fuses).
4. **Confidence gate.** Report only ≥8/10. Each finding must quote the motivating Evidence (pseudocode line, import, string, IPC channel). Everything else goes to an appendix as "verify".
5. **Report.** Table `Sev | Conf | Category | Finding | Evidence ID | Remediation`, mapped to OWASP/CWE where useful. End with: static review, not a penetration test. Vulnerabilities in third-party products go to the vendor's private channel, not public issues.

## Mode: research

1. **Question tree.** Split "how does X work" into sub-questions (data model, trigger, storage, network, UI). Mark which need artifact Evidence vs. public context.
2. **Public context first, cheaply.** Vendor docs, changelogs, patents, talks, open-source equivalents `[ext]` — to form hypotheses, not conclusions.
3. **Artifact Evidence.** Answer each sub-question with REA; independent sub-questions can run in parallel workers that each return Evidence IDs and open gaps.
4. **Triangulate.** A conclusion is strong when artifact Evidence and an independent source agree; when they conflict, the artifact wins for *this build* and the conflict is reported.
5. **Build.** If asked to reimplement, write original code from the behavioral spec — do not copy decompiled code or proprietary assets. Separate "observed behavior" from "our design choice".

## Mode: motion

Goal: extract the *parameters* of an interaction you admire and rebuild it with your own code.

1. Locate the component: `analyze_javascript_application` → search modules for `transition`, `animate`, `spring`, `cubic-bezier`, `keyframes`, `stiffness`, `damping`, `duration`, gesture handlers.
2. Record exact values (easing curves, spring stiffness/damping/mass, durations, delays, transform origins, thresholds) with Evidence IDs. For native apps, constants in `pseudocode` near the animation API calls.
3. Confirm with runtime when ambiguous: `capture_browser_scenario` performing the gesture, then compare timing.
4. Hand off to /emil-design-eng for judgment: is the motion purposeful, interruptible, reduced-motion safe, and does it fit *your* product? Rebuild with your own component code; values are facts, implementation is yours.

## Mode: firmware

1. **Scope.** Only images the user is licensed to analyze (own devices, support-provided images, vulnerability research under vendor policy). Many vendor images (including FortiOS) are encrypted or signed; do not attempt decryption-key extraction or signature bypass.
2. `inspect_firmware_regions` → `extract_firmware` into a new absolute directory (Linux host with Binwalk/Unblob, per REA's firmware guide). Inventory with `inspect_artifact`; open individual ELF services with `open_binary`.
3. For kernel modules use `inspect_kernel_module`; for packet behavior use recorded captures (`inspect_packet_capture`, `inspect_pcap_capture`) — REA does not capture live traffic.
4. Correlate component versions with published PSIRT advisories `[ext]`. For live FortiGate configuration, hardening and CLI remediation, switch to /fortinet-security-expert; REA covers the image, that skill covers the running device.

## Report

```
REA REPORT
════════════════════════════════════════
Question:     [what was asked]
Mode:         debug | audit | research | motion | firmware
Target:       [name, version, SHA-256, provider]
Answer:       [plain-language conclusion]
Observed:     [facts with Evidence IDs]
Inferred:     [claims with confidence and basis]
Unknown:      [bounded gaps and why]
External:     [ext] sources used
Next step:    [one concrete action]
Status:       DONE | DONE_WITH_CONCERNS | BLOCKED | NEEDS_CONTEXT
════════════════════════════════════════
```

Then close native sessions with `close_binary`, and `reset_investigation` with `save_path` when the evidence should be kept.
