# Portfolio package v1.0 — manifest

Every document here is a sanitized public edition, built from a versioned source and
checked by an automated gate before publication. The gate is fail-closed: a single
failed check blocks the upload.

**Package:** v1.2 · **Built:** 2026-09-11 · **Gate result:** 7/7 passed

| # | File | Version | Pages | SHA-256 |
|---|---|---|---:|---|
| 1 | `UDF2_ARCHITECTURE_v3.1-public.pdf` | v3.1 Public Edition | 26 | `09e1965f4fd584ec0db3ece6ed6df94c18954d53b555029b54284fac34316c50` |
| 2 | `AutoDoc_FW_ARCHITECTURE_v3-public.pdf` | v3 Public Edition | 26 | `cb2a179a1859d3526d61ebb0b6e71778e07b28fbae9ed13e89a9a31c9a04742d` |
| 3 | `FactoryFW_ASBUILT_v1-public.pdf` | v1 Public Edition | 15 | `b401a5e16f97fc40d953c0c162bf92bb086b1bb237ee8529b1fe6d42fda4c9dd` |
| 4 | `fleetguard-case-sheet-v2.pdf` | v2 | 3 | `9c3935aeb2f02796612526ebe54ae0e57928d683c879b7f49af10de4d08b7061` |
| 5 | `Deep-Research-Martins_v2.2-public-EN.pdf` | v2.1.2 + v2.2 wave 1 | 10 | `1517cb9458efcf12b07496a3c66c41c026f22a395bf4bc9efb36675dd571f26f` |
| 6 | `Deep-Research-Martins_v2.2-public.pdf` | v2.1.2 + v2.2 wave 1 (Czech) | 10 | `6c955ba19171dbf1541418924a9e2ca7b6ca923a922d8cc329eadedfb956ef0f` |
| 7 | `hyperv-recovery-case-sheet-v1.pdf` | v1 | 5 | `8222db19cd227be58585e88f91108ae35453d0ab46c883d8cdbf0882d4a77667` |

Verify any file with `shasum -a 256 <file>`.

## Pre-publication checks

| ID | Check | Fails when |
|---|---|---|
| K1 | File integrity | the PDF cannot be parsed, or is encrypted |
| K2 | Page format | any page deviates from A4 |
| K3 | Empty pages | any page carries less than 25 characters of text |
| K4 | Sanitization | any blocked client or project identifier appears in the text |
| K5 | Diagram rendering | unrendered diagram source is left in the text |
| K6 | Text layer | no text can be extracted (i.e. the file is a scan, not typeset) |

Two defects were found and fixed before this package was released: tall C4 diagrams
overflowed the A4 content box and left blank pages in two documents. Both were rebuilt
from source; content unchanged.

## Rules

1. A document enters the package only after passing all checks.
2. Any content change means a new document version and a new package version; hashes are recomputed.
3. Filenames carry the document version; the source file and its PDF share the same name.
