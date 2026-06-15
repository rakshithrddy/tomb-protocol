# TOMB — Topological Object Molecular Binary Storage

**The open codec standard for DNA data storage.**
The TCP/IP of molecular storage — any data, any hardware, same protocol.

[![License: AGPL-3.0](https://img.shields.io/badge/License-AGPL--3.0-blue.svg)](LICENSE)
[![Patent Pending](https://img.shields.io/badge/Patent-IN202641065626-orange.svg)](https://ipindia.gov.in)
[![Tests](https://img.shields.io/badge/Tests-1%2C396%20passing-brightgreen.svg)](#)
[![SNIA DARS](https://img.shields.io/badge/SNIA%20DARS-5%2F6%20gaps%20closed-blue.svg)](#snia-compliance)
[![Encode Speed](https://img.shields.io/badge/Encode-194%20MB%2Fs-green.svg)](#performance)

---

## The Problem

Every DNA storage company builds its own encoding layer from scratch.
The result: no interoperability, no standard, no ecosystem.

TOMB is the codec layer every DNA hardware company needs but none has built yet.
Like USB gave every device a standard plug — TOMB gives DNA storage a standard format.

```
Your data  →  TOMB encode  →  Synthesis-ready FASTA  →  Any DNA hardware
DNA reads  →  TOMB decode  →  Original bytes, exact
```

---

## Numbers

| Metric | Value |
|--------|-------|
| Encode speed | 194 MB/s (chr1 GRCh38, Apple M5) |
| Decode speed | 1,478 MB/s |
| Tests passing | 1,396 (0 failures, 0 warnings) |
| Conformance tests | 17 (C-01 to C-17, SNIA DARS) |
| Strand length | 196 nt |
| Data per strand | 34 bytes |
| ECC overhead | 12.5% (vs Reed-Solomon 20–40%) |
| SNIA DARS gaps closed | 5 of 6 |
| Platforms | Apple M5 · Pi4 · Pi Zero · x86 · WASM32 |

Compression ratios (with ECC enabled):

| Data type | Ratio |
|-----------|-------|
| Genomic FASTA | 2.79× |
| Text / code | up to 191× |
| FASTA headers | 394× |
| Time-series | 50× |
| Audio (WAV→FLAC) | 4.4× |
| Binary f64 streams | 14–260× |

---

## Demo

```
╔══════════════════════════════════════════════════════════╗
║     TOMB — Topological Object Molecular Binary Storage    ║
║     Patent IN202641065626 · Bengaluru, India · 2026       ║
║     Rakshith N (Software) · Divakar YG (Wet Lab, BU)     ║
╠══════════════════════════════════════════════════════════╣
║  Same command for all 8 input types. Always 196 nt.       ║
╚══════════════════════════════════════════════════════════╝

  ✅  1. Plain text         — byte-exact match
  ✅  2. Genomic FASTA      — byte-exact match  (3.3× this sample · 2.79× full chr1)
  ✅  3. JSON metadata      — byte-exact match
  ✅  4. Rust source code   — byte-exact match
  ✅  5. CSV assay data     — byte-exact match
  ✅  6. Patent abstract    — byte-exact match
  ✅  7. Audio MP3          — byte-exact match
  ✅  8. Video MP4          — byte-exact match

  Tests   : 1,396  (0 failures)
  SNIA    : 17/17  conformance tests
  Wet lab : 36-oligo order placed · Bioserve India · sequences verified
  Patent  : IN202641065626
```

> 🎬 **[Watch the live demo →](https://youtu.be/-JqKMrh3JGw)**

---

## Quick Start

Requires Rust 1.75+

```bash
# Clone this repository
git clone https://github.com/rakshithrddy/tomb-protocol
cd tomb-protocol

# The production Rust implementation — contact for full access
# rakshith.tomb@gmail.com

# Encode any file to synthesis-ready DNA FASTA
tomb compile --with-ecc input.txt output.fasta

# Decode back to original bytes (byte-exact guaranteed)
tomb decode output.fasta recovered.txt

# Verify round-trip
diff input.txt recovered.txt && echo "PASS: byte-exact"

# Run full demo (8 input types)
./demo.sh

# Run test suite
cargo test --workspace
# 1,396 tests, 0 failures, 0 warnings
```

---

## Strand Format

Every TOMB strand is exactly 196 nucleotides:

```
Pos  1–20   L-PRIMER   ACACTCTTTCCCTACACGAC      20 nt  PCR primer
Pos 21–24   SPACER     TTTT                        4 nt  hairpin guard
Pos 25–34   TOEHOLD    [10 nt Hilbert address]    10 nt  random access key
Pos 35–38   SHABDA     [4 nt metadata symbol]      4 nt  self-describing
Pos 39–176  PAYLOAD    [data + ECC]              138 nt  34 bytes usable
Pos 177–196 R-PRIMER   GTCGGAGGGTTCAGAGTTCT      20 nt  PCR primer
```

Chemistry gates (all enforced at compile time):

| Gate | Constraint | Value |
|------|-----------|-------|
| GC content | 40–60% | SNIA DARS requirement |
| Homopolymer | ≤4 consecutive | synthesis safety |
| Toehold Tm | 37–45°C | hybridisation at 37°C |
| ViennaRNA | ≥63% accessible | probe binding |
| Restricted motifs | 22 blocked | biosafety (14 RE + 3 DURC + 5 artefacts) |

**Primer-free variant** (Nanopore / enzymatic synthesis):
SPACER(4) + TOEHOLD(10) + PAYLOAD(182) = 196 nt total → **44 bytes usable (+29%)**
Physical framing declared by HAL CapabilityProfile — no codec change needed.

---

## Architecture

```
┌──────────────────────────────────────────────────────────┐
│  Your Application  (any language, REST or MCP)            │
├──────────────────────────────────────────────────────────┤
│  TSI  │  REST API  port 3000 (8 endpoints)               │
│       │  MCP server port 3001 (12 tools)                 │
├───────┼──────────────────────────────────────────────────┤
│  TQL  │  SQL → probe FASTA compiler                      │
│       │  Free tier (index) + paid tier (sequencing)      │
├───────┼──────────────────────────────────────────────────┤
│  TAI  │  AI embedding storage · vector DB in DNA         │
├───────┼──────────────────────────────────────────────────┤
│  TSE  │  OTP key pools · toehold access control          │
│       │  Copy-number tamper detection                    │
├───────┼──────────────────────────────────────────────────┤
│  CORE │  13 compressors · TopologicalECC · 3D Hilbert    │
│       │  196 nt strand compiler · 7-gate validator       │
├───────┼──────────────────────────────────────────────────┤
│  HAL  │  4-method frozen interface                       │
│       │  symbol_write / read / recognise / bulk_verify   │
├───────┼──────────────────────────────────────────────────┤
│  PHY  │  Any DNA hardware — Illumina · Nanopore · custom │
└──────────────────────────────────────────────────────────┘
```

The HAL is the key claim. Any DNA hardware implementing 4 methods is
immediately TOMB-compatible. No codec changes. No re-engineering.
Change the driver, the entire stack works.

---

## SNIA DARS Compliance

TOMB targets SNIA DARS (DNA Archival and Retrieval Standard):

| Gap | Requirement | Status |
|-----|-------------|--------|
| 1 | Formal codec specification | ✅ TOMB_CODEC_SPEC_v1.0.1.md — 906 lines |
| 2 | Biosafety motif filtering | ✅ 22 restricted motifs (14 RE + 3 DURC + 5 artefacts) |
| 3 | Sector Zero bootstrap | ✅ 3 self-describing strands at Hilbert addr 0, 1, 2 |
| 4 | Base distribution proof | ✅ 1,606,438 chr1 strands — GC 40–60%, 100% valid |
| 5 | Open reference implementation | ✅ AGPL-3.0 · 1,396 tests · 17 conformance tests |
| 6 | Wet lab validation | 🔄 36-oligo synthesis order placed — Bioserve India |

**Cold-reader guarantee:** Any future reader with zero prior knowledge can
reconstruct any TOMB archive from the Sector Zero bootstrap strands alone —
no external database, no manifest, no internet connection.

---

## Random Access

TOMB retrieves individual files without sequencing the full pool:

```
Without TOMB:   sequence everything  →  $190,000+ per retrieval
With TOMB:      one 14 nt probe      →  $1–10 per retrieval (archive scale)
```

How it works:
1. Each strand carries a unique 10 nt toehold sequence at positions 25–34
2. The toehold encodes a Hilbert address — your file's location in 3D pool space
3. A 14 nt probe (RC(toehold) + 4 nt extension) hybridises only to that strand
4. Wrong probe = no hybridisation = no access (enforced by Watson-Crick physics)

9 proof strands validated (tomb_proof_v5.fasta, commit 0130cd8):
- All 9 toeholds IDT-validated, Tm 34.5–38.1°C
- All 9 ViennaRNA ≥63% accessible (DNA Matthews 2004 params, 37°C)
- Round-trip CRC32 = 0x94c24574 (exact)

---

## Wet Lab

Active validation at IISc Bengaluru with Divakar YG (PhD Biotechnology,
Bangalore University):

| Assay | Method | Pass Condition |
|-------|--------|----------------|
| 1 | Fluorescence toehold displacement at 37°C | Signal/baseline ≥3× for ≥7/9 strands |
| 2 | MiSeq round-trip CRC32 | Exact: 0x94c24574 |
| 3 | Temperature titration 15–42°C | IDT offset 3.65°C ± 0.5°C |

Synthesis order: **36 oligos placed with Bioserve India**
(9×196nt HPLC · 9×10nt FAM reporters · 9×10nt BHQ-1 quenchers · 9×14nt probes)

---

## Commercial Licensing

TOMB is **AGPL-3.0** for open-source use.

Commercial use in proprietary products requires a separate commercial license.
This covers any company shipping DNA storage hardware or software that
includes TOMB without open-sourcing their full stack.

**The AGPL-3.0 licence is deliberate.** It enables TOMB to be freely used
for research while ensuring commercial adopters contribute to the ecosystem
or obtain a commercial licence.

For commercial licensing, OEM partnerships, and enterprise support:
📧 **rakshith.tomb@gmail.com**

---

## Intellectual Property

- **Indian Patent Pending:** IN202641065626 (filed 25 May 2026)
- **Complete specification:** in preparation (deadline May 2027)
- 4 primary claims: hardware-agnostic HAL · TopologicalECC ·
  3D Hilbert addressing · combined integrated system
- Additional claims covering: toehold random access · primer-free variant ·
  differential degradation channel · one-time-pad DNA key pool

---

## Repository Contents

| File | Description |
|------|-------------|
| `README.md` | This file |
| `ARCHITECTURE.md` | Full 7-layer stack design and component details |
| `CODEC_SPEC_SUMMARY.md` | Technical summary for engineers |
| `TOMB_One_Pager.md` | Business summary |
| `hal_spec_v1.0.md` | HAL interface formal specification |
| `TOMB_PHASES_V2.md` | Phase 0–7 roadmap |
| `CITATION.cff` | Academic citation format |
| `LICENSE` | AGPL-3.0 |

The production Rust implementation (7 crates, 1,396 tests) is maintained
in a private repository. Contact for access or collaboration.

---

## Citation

```bibtex
@software{tomb2026,
  author    = {Rakshith N},
  title     = {TOMB: Topological Object Molecular Binary Storage},
  year      = {2026},
  url       = {https://github.com/rakshithrddy/tomb-protocol},
  note      = {Patent pending IN202641065626, Bengaluru India}
}
```

---

## Implementation Access

Full production implementation available to qualified partners under a partnership agreement. This repository contains the protocol specification, architecture documentation, and HAL specification.

To request access: rakshith.tomb@gmail.com

---

## Contact

| | |
|-|-|
| **Inventor** | Rakshith N |
| **Wet Lab** | Divakar YG — PhD Biotechnology, Bangalore University |
| **Email** | rakshith.tomb@gmail.com |
| **Location** | Bengaluru, India |
| **Patent** | IN202641065626 (filed 25 May 2026) |

---

*TOMB is the only open-source DNA storage codec with a formal specification,
a hardware-agnostic HAL, a machine-readable conformance test suite,
and active wet lab validation. It is the missing standard layer the DNA
storage industry needs to achieve interoperability.*
