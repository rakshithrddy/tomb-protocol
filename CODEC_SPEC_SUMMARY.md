# TOMB CODEC SPEC v1.0.1 — Summary

Full specification: 906 lines, 17 conformance tests (C-01 to C-17)

## Strand Anatomy (Normative)

| Field | Position | Length | Value |
|-------|----------|--------|-------|
| L-PRIMER | 1–20 | 20 nt | ACACTCTTTCCCTACACGAC |
| SPACER | 21–24 | 4 nt | TTTT |
| TOEHOLD | 25–34 | 10 nt | Hilbert address (Prastara space: 688,128 sequences) |
| SHABDA | 35–38 | 4 nt | Self-describing metadata byte |
| PAYLOAD | 39–176 | 138 nt | Data + ECC (34 bytes usable) |
| R-PRIMER | 177–196 | 20 nt | GTCGGAGGGTTCAGAGTTCT |
| **TOTAL** | | **196 nt** | |

## Chemistry Gates (Normative)

All gates enforced at compile time. Any strand failing any gate is rejected.

| Gate | Constant | Value |
|------|----------|-------|
| STRAND_LEN | 196 | Exact strand length |
| GC_MIN | 0.40 | Minimum GC fraction |
| GC_MAX | 0.60 | Maximum GC fraction |
| MAX_RUN | 4 | Maximum homopolymer run |
| TM_MIN | 38.0°C | Minimum toehold melting temperature |
| TM_MAX | 50.0°C | Maximum toehold melting temperature |
| VIENNA_MIN | 0.63 | Minimum toehold unpaired fraction |

## ECC Parameters

- Algorithm: TopologicalECC XOR lattice (default) or RS(255,223) (SulbaKuttaka)
- Block size: 256 bytes
- Parity: 32 bytes per block (12.5% overhead)
- Corrects: substitutions · insertions · deletions · strand loss up to 12.5% (RS Kapha erasure mode, 32/255 symbols)

## Addressing

- Method: 3D Hilbert curve (Skilling's algorithm), order-8
- Address space: 2^24 = 16,777,216 per pool
- Toehold space: 688,128 valid sequences (Prastara validity space)
- Addresses 0, 1, 2: reserved for Sector Zero bootstrap

## Conformance

A TOMB-compatible implementation must pass all 17 tests in
TOMB_CODEC_SPEC_v1.0.1.md (tests C-01 through C-17).

Contact rakshith.tomb@gmail.com for the full specification and conformance suite.
