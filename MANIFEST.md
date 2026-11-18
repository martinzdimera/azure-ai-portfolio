# Portfolio package — manifest

Every document here is a sanitized public edition, built from a versioned source and
checked by an automated gate before publication. The gate is fail-closed: a single
failed check blocks the upload.

**Package:** v2.1 · **Built:** 2026-11-18 · **Gate result:** 10/10 PDF documents passed

| # | File | Version | Pages | SHA-256 |
|---|---|---|---:|---|
| 1 | `UDF2_ARCHITECTURE_REFERENCE_v4.0-public.pdf` | v4.0 Public Edition | 71 | `556c57d95e331bbb7125325c9d0fb1aba9719623c459a918343dcee249e4615d` |
| 2 | `AutoDoc_FW_ARCHITECTURE_v3-public.pdf` | v3 Public Edition (corrected edition) | 25 | `c1ffcc447543db32dd82a146ed849a0c8a12f27e09867601996549fa3ef44a5a` |
| 3 | `FactoryFW_ARCHITECTURE_REFERENCE_v2.0-public.pdf` | v2.0 Public Edition | 67 | `d0af2c7e30eb494c583ec3474fdd17518157699f68cb2f20a469a9dae7449b99` |
| 4 | `fleetguard-case-sheet-v2.4.pdf` | v2.4 | 3 | `408fc2939ca9b000bba564a29792da01fa3b274e12911586d9fcd6da725404db` |
| 5 | `Deep-Research-Martins_v2.2-public-EN.pdf` | v2.1.2 + v2.2 wave 1 | 10 | `1517cb9458efcf12b07496a3c66c41c026f22a395bf4bc9efb36675dd571f26f` |
| 6 | `Deep-Research-Martins_v2.2-public.pdf` | v2.1.2 + v2.2 wave 1 (Czech) | 10 | `6c955ba19171dbf1541418924a9e2ca7b6ca923a922d8cc329eadedfb956ef0f` |
| 7 | `hyperv-recovery-case-sheet-v1.1.pdf` | v1.1 | 5 | `dad7f20bb3b25bfb30f95c5d95e4b4865c30ddfe654ecbd0326f0e360b9afa18` |
| 8 | `hyperv-phase2-case-sheet-v1.0.pdf` | v1.0 (phase in progress) | 5 | `029e420558bbab60e885ec8182d672e0fd626bba7d7355a78b8d385878d41e91` |
| 9 | `fleetguard-delivery-mechanics-v1.0.pdf` | v1.0 Public Edition | 12 | `1e6dbcb3f443bfe999d3ee231eba5f0225c84db6aff59459fce67c370c20e600` |
| 10 | `Deep-Research-apparatus-reference-v3.0-public-EN.pdf` | v3.0 (English) | 15 | `4f2f507d6f4b52ba5b7dfab9f080fe0659af0193a0b708df485fc198d11401ee` |
| 11 | `mcp-azure-gateway-design/` | v1.0 design documentation (Draft), 62 files, 3862717 bytes | — | `index.html`: `3e6131a4dcaa59fef0de1616d737c1c8680a985fc2ff6424ea5fd2536e614e34`; file list: `088de0463ab365b4d081647404a34d8940247d4b6484e09f16a8f596bd1d29b5` |

Verify any file with `shasum -a 256 <file>`.
Verify the folder (file list) with `cd mcp-azure-gateway-design && find . -type f -print0 | LC_ALL=C sort -z | xargs -0 shasum -a 256 | shasum -a 256`.

Item 2 is a corrected edition and an exception to rule 2 below: a leftover internal editing note
on page 1 was removed and the document was rebuilt from source, which also regenerated layout and
diagram rendering (25 pages instead of 26); its version label stays v3. Item 11 is a folder of design
documentation published as written (Markdown, Mermaid/SVG diagrams, machine-readable contracts and
one HTML file). The PDF checks below do not apply to it; it was checked for client identifiers, and
its HTML file for externally loaded resources. In package v2.0 two lines of one document in
item 11 were edited to remove references to unpublished working files; the documentation version
label stays 1.0.

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
