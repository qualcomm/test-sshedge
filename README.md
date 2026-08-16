# test_sshedge — copyright/license checker fixtures

Deliberate test cases for the copyright-license-checker Action's **full-repo scan**
(`full_scan.py`). Each file encodes one scenario; the table below is the expected
`full_scan` outcome. The repo license resolves to **BSD-3-Clause-Clear**, so the
allowed set is the permissive licenses.

Run the comparison tool against this branch:

    python scripts/compare_tools_remote.py --repos qualcomm/test_sshedge

or the scanner directly on a local checkout:

    python full_scan.py qualcomm/test_sshedge false --repo-path <clone>

## Expected outcomes

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

Legend: **BLOCK** = blocking error (fails the scan); **WARN** = non-blocking warning;
**clean** = no finding.

Note: license *change* / *deletion* and the Qualcomm-copyright-rename allowance are
pull-request *diff* behaviors (`main.py`), not full-repo-scan behaviors, so they are
not represented in this repo.
