# test_sshedge — copyright/license checker fixtures

Deliberate test cases for the copyright-license-checker Action's **full-repo scan**
(`full_scan.py`). Each file encodes one scenario; the tables below are the expected
`full_scan` outcome. The repo license resolves to **BSD-3-Clause-Clear** (the `LICENSE`
file is effectively empty, so resolution falls through to the default), so the allowed
set is the permissive licenses.

## How it runs (GitHub Action)

Two workflows exercise the same scanner from different entry points:

- **`.github/workflows/full-repo-scan-direct.yml`** — calls the composite action directly
  (`qualcomm/copyright-license-checker-action/full-scan@full_scan`) with
  `fail_on_findings: false`. Report-only: the job stays green and the full per-file matrix
  shows in one report. This is the workflow to read.
- **`.github/workflows/full-repo-scan-test.yml`** — runs the scan through the shared
  `reusable-qcom-preflight-checks-orchestrator.yml` (`enable-full-repo-scan: true`). That
  path runs as a **gate**, so it exits non-zero (job fails) whenever a blocking fixture is
  present — expected here.

Locally, the identical run:

    python full-scan/full_scan.py qualcomm/test_sshedge false --repo-path <clone>

## Expected outcomes — core matrix

| File | SPDX in file | Copyright | full_scan | Why |
|------|--------------|-----------|-----------|-----|
| `clean.py` | BSD-3-Clause | Qualcomm | clean | compliant baseline |
| `compound_or_ok.py` | Apache-2.0 OR GPL-2.0-only | Qualcomm | clean | OR-group passes if any member is permissive |
| `build_no_header.mk` | (none) | (none) | clean | license-optional build file: missing header/copyright is OK |
| `incompatible.go` | GPL-2.0-only | Someone Else | BLOCK + WARN | copyleft under a permissive repo (INCOMPAT); holder not Qualcomm/LF (CRHOLDER) |
| `nolicense.py` | (none) | (none) | BLOCK | missing license header (NOLIC) + missing copyright (NOCR) |
| `build_incompatible.mk` | GPL-2.0-only | Qualcomm | BLOCK | build files are license-optional, but a *present* incompatible license still blocks |
| `proprietary_only.py` | LicenseRef-scancode-proprietary-license | Qualcomm | BLOCK | a lone proprietary license blocks |
| `compound_and_block.py` | GPL-2.0-only AND MIT | Qualcomm | BLOCK | AND-group requires every member to be permissive |
| `unknown.py` | LicenseRef-scancode-unknown-license-reference | Qualcomm | WARN | scancode cannot resolve the license (UNCERT) |
| `mixed_uncertain.py` | proprietary AND unknown | Qualcomm | WARN | proprietary mixed with other uncertain refs downgrades to a warning |
| `mit_janedoe.py` | MIT | Jane Doe | WARN | permissive license (clean) but holder not Qualcomm/LF (CRHOLDER) |
| `vendor/ignored.py` | GPL-2.0-only | Upstream Author | skipped by default | excluded by `.licenseignore` (`vendor/**`); scanned only with `--include-licenseignore`, then BLOCK |

## Expected outcomes — extended matrix (added for full coverage)

| File | SPDX in file | Copyright | full_scan | Case it covers |
|------|--------------|-----------|-----------|----------------|
| `bareword_noise.c` | BSD-3-Clause-Clear (+ "gpl/lgpl" in a comment) | Qualcomm | clean | bare-word noise filter: stray license words next to a real header are dropped (`confident_license_expression`) |
| `llvm_exception.cc` | Apache-2.0 WITH LLVM-exception (SPDX id) | Qualcomm | clean | `WITH` exception expression accepted; also covers `.cc` |
| `llvm_exception_text.cc` | Apache-2.0 WITH LLVM-exception (full LLVM notice) | Qualcomm | clean | scancode maps the LLVM notice text to the same `WITH` expression |
| `clean.rs` | BSD-3-Clause-Clear | Qualcomm | clean | `.rs` source coverage (repolinter-parity extension) |
| `boot.S` | BSD-3-Clause-Clear | Qualcomm | clean | `.S` assembly coverage (matched exact-case) |
| `lower.s` | GPL-2.0-only | Someone Else | not scanned | `.s` is **not** in the source list (`str.endswith` is case-sensitive) — a GPL file that escapes the scan; parity gap to note |
| `mit_nocopyright.py` | MIT | (none) | BLOCK | isolates the missing-copyright check: permissive license, no copyright (NOCR) |
| `lf_ok.py` | BSD-3-Clause-Clear | Linux Foundation 2020 | clean | Linux-Foundation holder inside the intentional 2012–2022 window |
| `lf_capped.py` | BSD-3-Clause-Clear | Linux Foundation 2024 | WARN | LF year cap: 2023+ falls outside the expected pattern (CRHOLDER) |
| `recipe.bb` | GPL-2.0-only | Upstream | not scanned | BitBake `.bb` excluded from the scan entirely |
| `helper.bbclass` | GPL-2.0-only | (none) | not scanned | `.bbclass` excluded (repolinter *does* scan it — known parity gap) |
| `soong_incompatible.bp` | GPL-2.0-only | Qualcomm | BLOCK | `.bp` is license-optional, but a *present* incompatible license still blocks |
| `soong_no_header.bp` | (none) | (none) | clean | `.bp` license-optional: missing header/copyright is OK |
| `data.json` | GPL-2.0-only (as a string) | — | not scanned | `.json` is an excluded extension |
| `config.yml` | GPL-2.0-only | — | not scanned | `.yml` is an excluded extension |
| `or_then_and_block.py` | (MIT OR Apache-2.0) AND GPL-2.0-only | Qualcomm | BLOCK | full-scan evaluates **every** AND group — the trailing GPL fails even though the leading OR is permissive |
| `and_group_in_or.py` | (MIT AND BSD-3-Clause) OR GPL-2.0-only | Qualcomm | BLOCK (**should be clean**) | **known false positive**: the OR's left branch is fully permissive, but the string-split evaluator mis-parses a parenthesized `(A AND B) OR C` |

## Expected outcomes — non-SPDX license markings

These carry a real license **notice text** (or none) instead of an
`SPDX-License-Identifier:` tag, so they exercise scancode's text-matching path
(`2-aho`/`3-seq`) and the `confident_license_expression` length filter rather than the
`1-spdx-id` matcher.

| File | Marking in file | Copyright | full_scan | Case it covers |
|------|-----------------|-----------|-----------|----------------|
| `bsd3_notice.c` | BSD-3-Clause full "Redistribution and use…" notice | Qualcomm | clean | permissive license recognized from notice text, no SPDX tag |
| `mit_notice.py` | MIT "Permission is hereby granted…" notice | Qualcomm | clean | permissive MIT from text |
| `apache_notice.py` | Apache-2.0 "Licensed under the Apache License, Version 2.0…" notice | Qualcomm | clean | permissive Apache-2.0 from text |
| `gpl_notice.c` | GPL "This program is free software…" notice | Someone Else | BLOCK + WARN | copyleft detected from text (`GPL-2.0-or-later`, INCOMPAT) + non-Qualcomm holder (CRHOLDER) |
| `proprietary_notice.py` | "All rights reserved. Confidential and proprietary." | Qualcomm | BLOCK | proprietary detected from a 3-token notice (kept at `MIN_CONFIDENT_MATCH_LENGTH`) → lone proprietary blocks |
| `license_ref.c` | "Licensed under the terms described in the LICENSE file…" | Qualcomm | WARN | a referenced-license phrase → `LicenseRef-scancode-unknown-license-reference` (UNCERT) |
| `copyright_only.py` | (none — copyright line only) | Qualcomm | BLOCK | copyright present but no license notice → No license header found (NOLIC); complements `mit_nocopyright.py` (the inverse) |

Legend: **BLOCK** = blocking error (fails the gate); **WARN** = non-blocking warning;
**clean** = no finding; **not scanned** = filtered out before scanning (excluded/uncovered
extension or `.licenseignore`).

## Notes

- License *change* / *deletion* and the Qualcomm-copyright-rename allowance are pull-request
  *diff* behaviors (`main.py`), not full-repo-scan behaviors, so they are not represented here.
- `and_group_in_or.py` is the one intentional **false positive** in this set: the scanner's
  hand-rolled SPDX evaluator splits on ` AND ` before honoring the outer parentheses, so a
  fully-permissive `(A AND B)` branch inside a top-level `OR` is wrongly rejected. It is kept
  as a regression marker for the parser fix (evaluate via a real SPDX expression parser).
