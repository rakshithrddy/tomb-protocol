# TOMB Architecture

This document describes the internal design of every layer of the TOMB protocol stack — the data structures, algorithms, protocols, and invariants that hold the system together. It is written for engineers who need to understand, extend, implement against, or port TOMB to new hardware.

**Patent Pending — IN202641065626**

---

## Contents

1. [Overview: The Layered Protocol Stack](#1-overview-the-layered-protocol-stack)
2. [Layer Diagram](#2-layer-diagram)
3. [The HAL Contract — The Most Important Design Decision](#3-the-hal-contract--the-most-important-design-decision)
4. [Data Flow: Encode Path](#4-data-flow-encode-path)
5. [Data Flow: Decode Path](#5-data-flow-decode-path)
6. [The Strand as the Unit of Storage](#6-the-strand-as-the-unit-of-storage)
7. [Addressing: 3D Hilbert Curve](#7-addressing-3d-hilbert-curve)
8. [Compression: Two-Stage Pipeline](#8-compression-two-stage-pipeline)
9. [ECC: Topological Lattice](#9-ecc-topological-lattice)
10. [Molecular Compiler: Rejection Sampling](#10-molecular-compiler-rejection-sampling)
11. [Phase Roadmap](#11-phase-roadmap)
12. [SNIA DARS Conformance](#12-snia-dars-conformance)
13. [Protocol Versioning](#13-protocol-versioning)
14. [Cross-References](#14-cross-references)

---

## 1. Overview: The Layered Protocol Stack

TOMB is not a single algorithm. It is a **layered protocol stack** — a set of precisely defined interfaces and behaviors, where each layer has exactly one responsibility and communicates with adjacent layers through a frozen interface.

This architecture is directly analogous to the TCP/IP network stack:

```
TCP/IP analogy          TOMB equivalent
──────────────────────────────────────────
Application (HTTP)  →   Application (any software using TOMB)
Transport (TCP)     →   tomb-tsi (Standard Interface)
Network (IP)        →   tomb-tql / tomb-tmc (Query, Compute)
Data Link           →   tomb core (Codec + ECC + Addressing)
Physical            →   HAL + Drivers (Hardware Abstraction)
```

The key insight is the same as it was for TCP/IP in 1973: **if you freeze the interface between layers, the layers below can evolve independently of the layers above**. When a new DNA synthesizer appears, you write a new driver. The application that stores photos never changes. The protocol that encodes those photos never changes. Only the driver changes — and it is bounded by the 4 frozen HAL methods.

### Design principles

1. **The HAL is the boundary.** Nothing above the HAL ever touches hardware. Nothing below the HAL ever knows about encoding, compression, or ECC.

2. **Every algorithm is a plugin.** Compression is a trait (`TombCompressor`). ECC is a trait (`TombECC`). Hardware is a trait (`TOMBHardwareDriver`). No algorithm is hardcoded where a trait should be.

3. **The strand structure is frozen.** 196 nucleotides. Primer (20) + Spacer (4) + Toehold (10) + Payload (142) + Primer (20). Field boundaries do not change without a major version bump.

4. **Backwards compatibility is a protocol guarantee.** Every strand encodes its version. A decoder MUST be able to read strands from any previous minor version. A decoder SHOULD attempt to read strands from any previous major version.

5. **Conformance first.** A feature is not part of the TOMB protocol unless it has a test in the conformance suite. The conformance suite is what makes TOMB a standard, not just a program.

6. **No wet-lab assumptions in software.** The core encoding logic assumes perfect synthesis. Hardware-specific noise, error rates, and quirks belong in the driver's `ErrorProfile`, not in the codec.

---

## 2. Layer Diagram

```
┌─────────────────────────────────────────────────────────────────────────┐
│  L6  APPLICATION LAYER                                                   │
│      Any software that reads and writes data                             │
│      (file systems, databases, AI models, web apps)                     │
├─────────────────────────────────────────────────────────────────────────┤
│  L5  SECURE ENCLAVE — tomb-tse                                           │
│      Molecular OTP | Toehold-gated access control                        │
│      Copy-number integrity | Quantum-proof key storage                   │
├─────────────────────────────────────────────────────────────────────────┤
│  L4  AI INTEGRATION — tomb-tai                                           │
│      Embedding store | Model archival | DNA vector database              │
│      Semantic similarity search over DNA-stored embeddings               │
├─────────────────────────────────────────────────────────────────────────┤
│  L3  DISTRIBUTED PROTOCOL — tomb-tdp                                     │
│      Federated pool network | Sharded routing | Consensus monitor        │
│      Cross-pool query executor | Health and repair                       │
├─────────────────────────────────────────────────────────────────────────┤
│  L2  QUERY + COMPUTE — tomb-tql / tomb-tmc                               │
│      TQL: SQL-like query language over DNA pools                         │
│      MISA: molecular instruction set for in-DNA computation              │
│      Probe compiler | Execution plan | Cost model                        │
├─────────────────────────────────────────────────────────────────────────┤
│  L1  STANDARD INTERFACE — tomb-tsi                                       │
│      REST API (port 3000) | MCP server (port 3001)                       │
│      POSIX-like file interface | Shared application state                │
├─────────────────────────────────────────────────────────────────────────┤
│  L0  CORE PROTOCOL — tomb (Phase 0)                                      │
│  ┌──────────────────────────────────────────────────────────────────┐   │
│  │  MOLECULAR COMPILER                                               │   │
│  │  bytes → 196-nt synthesis-ready FASTA strands                    │   │
│  │  Rejection sampling | span_fold | GC + homopolymer constraints   │   │
│  ├──────────────────────────────────────────────────────────────────┤   │
│  │  HILBERT ADDRESSING          │  TOPOLOGICAL ECC                  │   │
│  │  3D Skilling curve           │  √n × √n lattice parity           │   │
│  │  40-bit, bijection proven    │  12.5% overhead, all 4 error types│   │
│  ├──────────────────────────────────────────────────────────────────┤   │
│  │  TOPOLOGICAL SYMBOL ENCODER  │  FRACTAL DELTA COMPRESSION        │   │
│  │  256-symbol knot alphabet    │  Multi-scale delta + RLE          │   │
│  │  4× Shannon capacity         │  Entropy-gated, Rayon parallel    │   │
│  ├──────────────────────────────────────────────────────────────────┤   │
│  │  NUCLEOTIDE PACKER           │  TOMB INDEX                       │   │
│  │  2-bit ACGT | N-run RLE      │  Named file storage, CRC32        │   │
│  │  FASTA-aware auto-detect     │  O(1) lookup, 3 replicas          │   │
│  ├──────────────────────────────────────────────────────────────────┤   │
│  │  HAL — Hardware Abstraction Layer                                 │   │
│  │  FROZEN: symbol_write | symbol_read | symbol_recognise | bulk_verify │
│  └──────────────────────────────────────────────────────────────────┘   │
├─────────────────────────────────────────────────────────────────────────┤
│  HARDWARE DRIVERS (below the HAL)                                        │
│  SoftwareSimulatorDriver (DNA / memristor / NVMe profiles)               │
│  Future: NanoporeDriver | TwistDriver | MemristorDriver                  │
└─────────────────────────────────────────────────────────────────────────┘
```

---

## 3. The HAL Contract — The Most Important Design Decision

### What the HAL is

The Hardware Abstraction Layer (HAL) is the single interface point between TOMB's encoding logic and any physical storage hardware. Think of it as the USB standard for molecular storage: just as any USB device works with any USB port regardless of the internal implementation, any hardware that implements the 4 HAL methods works automatically with the entire TOMB protocol stack.

The HAL is defined by the `TOMBHardwareDriver` trait. Every hardware implementation MUST implement exactly these 4 core methods:

| Method | Signature | What it does |
|--------|-----------|-------------|
| `symbol_write` | `(addr: HilbertAddress3D, sym: TopologicalSymbol, verify: bool) → WriteResult` | Write one topological symbol to a 3D address |
| `symbol_read` | `(addr: HilbertAddress3D, num_reads: usize) → ReadResult` | Read one symbol from a 3D address, with confidence score |
| `symbol_recognise` | `(raw_signal: &[f32]) → RecogniseResult` | Classify a raw hardware signal into a topological symbol |
| `bulk_verify` | `(addresses: &[HilbertAddress3D], expected: &[TopologicalSymbol]) → VerifyResult` | Verify stored symbols match expected values |

### Why freezing the interface is what makes an open standard

An interface that can change is not an interface — it is a suggestion. The moment a hardware vendor ships a product and the HAL changes, the vendor must update their driver, retest, and reshipping. Every application that depended on that driver must be retested. The ecosystem fragments.

TCP/IP solved this in 1973 by freezing the packet format. The packet format has not changed in 50 years. Every network device built in those 50 years speaks the same language. TOMB applies the same principle to molecular storage.

The 4 HAL methods are chosen with the following properties:

- **Minimal**: They represent the smallest possible set of operations that gives the full TOMB capability. No operation can be removed without breaking something. No operation needs to be added to support any known storage medium.
- **Hardware-neutral**: The method signatures use `TopologicalSymbol` and `HilbertAddress3D`, not DNA bases, resistance levels, or voltage thresholds. Hardware-specific details are internal to the driver.
- **Capability-driven**: At mount time, the HAL reads a `CapabilityProfile` from the hardware. TOMB auto-tunes all parameters (ECC redundancy, parallelism, read retry count) based on the reported error profile and capacity. Hardware does not need to be re-encoded when it is replaced with a more capable model.

### What happens when new hardware arrives

When a DNA synthesizer or nanopore sequencer with new capabilities arrives:

1. An engineer writes a new struct that implements `TOMBHardwareDriver`
2. They implement the 4 methods using the new hardware's SDK
3. They fill in a `CapabilityProfile` with the hardware's error rates and capacity
4. They register the driver

No encoding code changes. No application code changes. No data needs to be migrated. This is the direct equivalent of plugging a new hard drive into a computer — the OS does not change, applications do not change, only the driver changes.

### The HAL MUST NOT be violated

The rule is explicit in the codebase: nothing in layers L1–L6 MUST call anything on hardware directly. Nothing in the hardware driver MUST know about topological symbols, Hilbert addresses, or ECC. The HAL is the only crossing point. Violating this rule breaks the open standard guarantee.

---

## 4. Data Flow: Encode Path

The following traces a 1 KB (1,024 byte) input through each stage, showing approximate byte counts at each transformation.

### Input

```
Input: 1,024 bytes of arbitrary data
```

### Stage 1: NucleotidePacker

The packer inspects the input for FASTA/FASTQ signatures. For non-genomic data, it passes through unchanged. For genomic data (ACGTN sequences), it applies 2-bit packing and N-run RLE.

```
Non-genomic 1 KB input → 1,024 bytes (passthrough)
Genomic 1 KB input     → ~256 bytes (4:1 ratio, 2-bit packing)
```

### Stage 2: FractalDeltaEncoder

The encoder operates in 64 KB chunks (non-genomic) or 256 KB chunks (genomic). For 1 KB input, the entire input fits in one chunk. The encoder:
- Tries 5 delta scales (1, 2, 4, 8, 16) and picks the best
- RLE-compresses the delta stream
- Prepends a 1-byte mode header (`0x01` = compressed, `0xFE` = raw)
- Entropy gate: if result ≥ 97.5% of input, writes raw with `0xFE` header

```
Highly structured 1 KB → ~12 bytes compressed (84× ratio)
Random 1 KB            → 1,025 bytes (1 raw header byte + passthrough)
Genomic 1 KB           → ~256 bytes compressed (4× ratio)
```

### Stage 3: TopologicalSymbol Encoding

Each byte maps to a `TopologicalSymbol` via a fixed bijection. This stage adds no overhead — it is a lossless representation change. The symbol count equals the byte count.

```
N bytes → N TopologicalSymbol values (1:1 mapping)
```

### Stage 4: TopologicalECC

The ECC stage arranges symbols in a √n × √n grid. For n symbols, it appends √n row-parity symbols and √n column-parity symbols. Total symbols after ECC = n + 2√n.

```
1,024 symbols → grid 32×32
→ 32 row parity symbols + 32 column parity symbols = 64 parity symbols
→ 1,088 symbols total
→ Overhead: 64/1024 = 6.25% per dimension, 12.5% combined
```

Parallel: each ECC block is independent and runs on its own Rayon thread.

### Stage 5: HilbertAddress3D

Each ECC block receives a unique 3D Hilbert address. The address is derived from a linear index using Skilling's inverse: `(x, y, z) = hilbert_to_xyz(index, order=8)`.

```
Block N → address (x, y, z) where xyz_to_hilbert(x,y,z,8) = N
Each address is unique: bijection proven
```

### Stage 6: MolecularCompiler

Each block's payload symbols are encoded as a 142-nucleotide DNA sequence via rejection sampling with an LCG seed. The compiler assembles the full 196-nt strand:

```
1 block of 1,088 symbols →
  splits into strands of 142 symbols each
  = ceil(1088/142) = 8 strands
  each strand: 196 nucleotides
  = 8 × 196 = 1,568 nucleotides of FASTA output

Each strand accompanies a 14-nt probe strand:
  + 8 × 14 = 112 nucleotides of probe FASTA
```

### Stage 7: FASTA output

```
Final output for 1 KB input:
  ~8 data strands × 196 nt = ~1,568 nt
  ~8 probe strands × 14 nt = ~112 nt
  FASTA headers (metadata): ~8 × 80 chars = ~640 chars
  Total FASTA file: ~2.3 KB
```

The FASTA expansion factor (~2.3× for small inputs) is inherent in the transition from digital bits to physical molecules. At scale (240 MB chromosome), the compression pipeline dominates and the ratio improves significantly.

---

## 5. Data Flow: Decode Path

The decode path is the strict inverse of the encode path. TOMB's bijection guarantee at every stage ensures perfect reconstruction.

### Stage 1: FASTA parsing

Parse each FASTA record. Extract from the header:
- `addr=<N>` — the Hilbert index
- `seed=<N>` — the LCG seed offset used during encoding
- `symbols=<N>` — number of payload symbols in this strand
- `valid=true/false` — synthesis validation status

### Stage 2: Sort by Hilbert address

Re-sort all parsed strands by their `addr` field (Hilbert index). This restores the original block ordering, regardless of the order the sequencer returned them.

```
FASTA records (unordered by sequencing) →
sorted list by hilbert_index ascending →
original block order restored
```

### Stage 3: DNA-to-symbol reconstruction

For each strand, apply the LCG seed (from the `seed` header field) to reconstruct the symbol-to-nucleotide mapping used during encoding. This is the inverse of the rejection-sampling step. The result is the original sequence of `TopologicalSymbol` values.

### Stage 4: TopologicalECC correction

Re-assemble the symbol grid. Recompute row and column parities. If any parity check fails:
- Identify the row and column that both fail: their intersection is the errored symbol
- Reconstruct the correct symbol value from the row parity XOR
- Correct in place

After ECC correction, strip parity symbols to recover the original n data symbols.

### Stage 5: TopologicalSymbol to bytes

Apply `symbol.to_byte()` to each symbol. The bijection is exact: the reconstructed byte sequence is identical to the input to Stage 3 of the encode path.

### Stage 6: FractalDeltaDecoder

Inspect the mode header byte:
- `0x01` (compressed): apply the inverse delta and RLE decoding sequence
- `0xFE` (raw): strip the header byte, return the rest unchanged
- `0x02` (parallel): decompose into chunks, decode each independently, concatenate

### Stage 7: NucleotideUnpacker

If genomic data was detected during encoding (signaled by a header flag in the packed format), unpack the 2-bit representation back to ACGT characters. Reconstruct N-runs from RLE spans.

### Stage 8: Output

Write the reconstructed bytes to the output file. The result MUST be byte-for-byte identical to the original input. Zero corruption. Zero loss.

---

## 6. The Strand as the Unit of Storage

### Why 196 nucleotides?

Think of a strand as a sentence in a book. If the sentence is too short, you waste most of it on punctuation (primers). If it is too long, it becomes harder to synthesize reliably and more expensive per base. 196 nucleotides is the current sweet spot:

- Twist Bioscience and IDT both offer reliable synthesis up to 200–300 nt
- The 20 nt primers are standard Illumina sequencing primers (proven in millions of experiments)
- 142 nt payload gives enough room for meaningful data density per strand
- Total synthesis cost per byte is minimized at this length

### Field layout (positions are 1-indexed)

```
Position    1–20:   LEFT PRIMER    (20 nt)
Position   21–24:   SPACER         (4 nt, always TTTT)
Position   25–34:   TOEHOLD        (10 nt)
Position   35–176:  PAYLOAD        (142 nt)
Position  177–196:  RIGHT PRIMER   (20 nt)
─────────────────────────────────────────
Total: 196 nucleotides
```

### Field-by-field design rationale

**Left primer (positions 1–20: `ACACTCTTTCCCTACACGAC`)**

A primer is a short, known sequence that DNA polymerase uses as a starting point for PCR amplification. The left primer serves two purposes: first, it enables PCR amplification (exponentially copying the strand from a pool of millions of molecules for sequencing); second, positions 19–20 encode the TOMB protocol version (see Section 13).

The primer sequence is chosen to satisfy the same synthesis constraints as the rest of the strand: GC content in range, no homopolymer runs, no self-complementarity. It is a fixed constant across all strands in a pool, which means it cannot be used to distinguish one strand from another — only the toehold does that.

**TTTT spacer (positions 21–24)**

The spacer is four thymine (T) bases inserted between the primer and the toehold. Its purpose is structural: it physically separates the primer sequence from the toehold so that the toehold does not fold into the primer and become inaccessible.

Before the spacer was added, ViennaRNA thermodynamic analysis showed that the toehold was buried in a hairpin structure formed with the adjacent primer sequence. The toehold bases (positions 25–34) appeared as paired dots in the dot-bracket notation instead of unpaired dots — meaning no probe could bind to them. The TTTT spacer breaks this hairpin. After adding it, ViennaRNA confirms positions 25–34 are unpaired (accessible) at 37°C physiological conditions.

The spacer MUST NOT be removed, MUST NOT be changed, and MUST NOT be moved.

**Toehold (positions 25–34)**

Think of the toehold like a barcode on a package. It identifies which address (shelf location) this strand belongs to, and it is the binding site for selective retrieval probes.

The toehold encodes the Hilbert address: each of the 10 nucleotides encodes 2 bits (A=00, C=01, G=10, T=11), giving 20 bits of addressing information (1,048,576 unique toehold sequences at order=8). The toehold is also the binding site for the 14-nt companion probe strand.

The toehold must be:
- Thermodynamically accessible: positions 25–34 must be unpaired in the minimum free energy structure
- High enough Tm to bind probes: ≥ 40°C under standard conditions
- Low enough GC to avoid overly stable secondary structure: GC ≤ 60%

**Payload (positions 35–176)**

The payload carries the actual data: topological symbols encoded as DNA bases using an LCG (Linear Congruential Generator) seed that is recorded in the FASTA header. 142 bases at 2 bits per base = 284 bits = 35.5 bytes of raw DNA capacity per strand, which after the topological symbol encoding (which packs 8 bits per symbol into the sequence via the rejection-sampling/LCG system) carries the data.

**Right primer (positions 177–196: `GTCGGAGGGTTCAGAGTTCT`)**

The reverse complement of a standard Illumina sequencing primer. Enables amplification and sequencing from the opposite end. Also enables paired-end sequencing, which provides error correction at the sequencing stage.

---

## 7. Addressing: 3D Hilbert Curve

### Why a space-filling curve?

A DNA pool is physically three-dimensional: strands are distributed in a volume of solution. When you retrieve strands, you want to retrieve strands that are logically adjacent (data from the same file) to also be physically adjacent (in the same region of the pool). This minimizes the volume you need to scan.

A Hilbert curve is a mathematical object that "fills" a multidimensional space in a way that preserves locality: points that are close together on the 1D index are also close together in the 3D space. It is like a very efficient filing system where related files are always stored near each other.

### Why 3D?

DNA pools are inherently 3D objects — you can address them by x, y, z coordinates in a microfluidic chip, a tube, or a voxel array. A 3D Hilbert curve maps a single linear address (the strand index) to a unique (x, y, z) coordinate triplet. This mapping is:

- **Surjective**: every grid voxel is assigned at least one address (100% space utilization, no wasted positions)
- **Injective**: no two linear indices map to the same (x, y, z) triplet (bijection)
- **Locality-preserving**: strands with sequential addresses are in adjacent voxels

### Skilling's algorithm

TOMB uses Skilling's algorithm (AIP Conf. Proc. 707, 381, 2004) for the 3D Hilbert curve. This is a provably correct, efficient implementation for arbitrary dimension and order. The implementation is in `tomb/src/addressing/hilbert3d.rs`.

### Address space

At Hilbert order 8 (the current default):
```
Capacity = 8^3 per order × 8 orders = 2^24 addresses per axis... 
Actually: capacity = (2^order)^3 = (2^8)^3 = 2^24 = 16,777,216 addresses at order=8
Maximum order supported: 21 (limited by u64: 2^(3×21) < 2^64)
```

The 10-nt toehold encodes a 20-bit Hilbert index (values 0–1,048,575). The 40-bit address space referenced in the system overview refers to the full Hilbert space at higher orders available by changing the compiler configuration. The toehold encoding is the limiting factor for the default configuration.

### What order 8 means

Order 8 means the grid is 256 × 256 × 256 (each axis divided into 256 segments). The Hilbert curve threads through all 16,777,216 cells of this grid exactly once, then stops. Increasing the order to 9 gives a 512 × 512 × 512 grid with 134 million addresses, and so on.

The `HilbertAddress3D::capacity(order)` function returns the total address count for any given order. The compiler's `hilbert_order` configuration field controls this at runtime.

---

## 8. Compression: Two-Stage Pipeline

### Why two stages?

Different data types have different compressibility. A photograph is already compressed. Human genomic sequence is 75% predictable from context. Source code is highly structured. Random binary data is not compressible at all. A single compression algorithm cannot be optimal for all of these.

TOMB's two-stage pipeline handles each data type correctly:

**Stage 1: NucleotidePacker** — handles genomic data specifically
**Stage 2: FractalDeltaEncoder** — handles structured/repetitive data generally

The entropy gate ensures that neither stage ever makes the data larger: if compression does not help, the raw data is passed through with a single-byte mode flag.

### Stage 1: NucleotidePacker

This stage activates automatically when the input data is detected to be genomic (FASTA or FASTQ format, or a raw ACGTN sequence). It applies two transformations:

**2-bit packing**: The four DNA bases (A, C, G, T) are encoded as 2-bit values (A=00, C=01, G=10, T=11). Four bases pack into one byte. This gives a 4:1 compression ratio on pure ACGT data.

**N-run RLE (Run-Length Encoding)**: Genomic sequences contain long runs of 'N' (unknown base). Human chromosome 1 contains approximately 30 million consecutive N's in centromeric and telomeric regions. Storing these naively would require 30 million bytes. TOMB's N-run RLE encodes each run as a (position, length) span, reducing the entire telomeric/centromeric region to approximately 50 spans — about 800 bytes instead of 30 megabytes.

**FASTA header preservation**: FASTA files include description lines starting with `>`. The packer detects and preserves these headers so that the original FASTA file can be reconstructed exactly, including sequence names and descriptions.

### Stage 2: FractalDeltaEncoder

This stage handles structured/repetitive data by encoding differences (deltas) between values rather than values themselves. Think of it like a time series chart: instead of recording the absolute value at each time point, you record how much it changed from the previous point. If the data is smooth, the changes are small and compressible.

The encoder is "fractal" because it tries multiple delta scales:
- **Scale 1**: delta between adjacent bytes (good for smooth gradients)
- **Scale 2**: delta between bytes 2 apart (good for alternating patterns)
- **Scale 4, 8, 16**: larger scales for longer-range structure

The encoder picks the scale that produces the smallest delta values, then applies RLE to collapse runs of identical delta values.

**Entropy gate**: Before writing compressed output, the encoder checks whether the compressed size is actually smaller than the input. If not (e.g., for truly random data), it writes the raw input with a single-byte `0xFE` mode header. This means TOMB never makes data larger.

**Rayon parallelism**: The encoder splits input into 64 KB chunks (non-genomic) or 256 KB chunks (genomic) and processes each chunk independently on a Rayon thread pool. Each chunk includes its own mode header so it can be decoded independently.

### Interaction between the two stages

```
Input type          Stage 1 (Nucleotide)    Stage 2 (FractalDelta)    Net result
───────────────────────────────────────────────────────────────────────────────
Genomic FASTA       4:1 packing + N-RLE     4× additional             ~4× total
Structured text     passthrough             84× on ideal data          84× peak
Random binary       passthrough             entropy gate (raw)         1× (no loss)
Compressed media    passthrough             entropy gate (raw)         1× (no loss)
```

---

## 9. ECC: Topological Lattice

### Why ECC for DNA?

DNA synthesis and sequencing are imperfect. Every synthesizer introduces errors. Every sequencer introduces errors. Unlike magnetic storage (which has well-characterized substitution errors), DNA can suffer four distinct error types:

1. **Substitution**: one base becomes another (A→G, etc.)
2. **Insertion**: an extra base is inserted
3. **Deletion**: a base is missing
4. **Strand loss**: an entire strand is lost from the pool (never sequenced)

Traditional ECC codes like Reed-Solomon are designed for substitution errors only. They do not handle insertions, deletions, or strand loss well. TOMB's TopologicalECC is designed from first principles for all four DNA error types.

### The √n × √n lattice

Think of a crossword puzzle grid. Each cell holds one symbol. The ECC adds one extra row and one extra column of "checksum" cells: the XOR sum across each row (in the extra column) and the XOR sum down each column (in the extra row).

```
Data symbols arranged in a √n × √n grid:

    col0  col1  col2  ... col(√n-1) | col_parity
    ───────────────────────────────────────────
row0│ s00   s01   s02  ...  s0k    | p0c  (XOR of row 0)
row1│ s10   s11   s12  ...  s1k    | p1c  (XOR of row 1)
row2│ s20   s21   s22  ...  s2k    | p2c  (XOR of row 2)
 .  │  .     .     .   ...   .     |  .
rowm│ sm0   sm1   sm2  ...  smk    | pmc  (XOR of row m)
────┼───────────────────────────────────────────
row │ pc0   pc1   pc2  ...  pck    |      (XOR of each col)
parity
```

**Error correction**: If exactly one symbol is wrong:
- Its row parity check fails (the XOR of that row no longer matches `p_rc`)
- Its column parity check fails (the XOR of that column no longer matches `p_cc`)
- The intersection of the failing row and failing column identifies the wrong symbol
- The correct value is computed as: `correct = row_parity_symbol XOR all_other_row_symbols`

**Error detection**: Two symbol errors are detected (the parities fail) but cannot be corrected without additional information.

**Strand loss**: If an entire strand is lost, that corresponds to one row being missing. The row can be reconstructed from the column parities: for each column position, the lost value = column_parity XOR all_other_row_values_in_that_column.

### Why 12.5% overhead beats Reed-Solomon

For a √n × √n grid with n data symbols:
- Row parity symbols: √n
- Column parity symbols: √n
- Total parity: 2√n
- Overhead: 2√n / n = 2/√n

For n = 256 (16×16 grid): overhead = 2/16 = 12.5%
For n = 1024 (32×32 grid): overhead = 2/32 = 6.25%

Reed-Solomon for DNA typically operates at 20–40% overhead because it must handle the higher error rates of biological synthesis, the variable-length nature of insertions/deletions, and strand loss. TOMB's lattice scheme achieves lower overhead by exploiting the 2D structure of the error space.

### Tiled computation for cache efficiency

The parity computation uses a tiled pass over the symbol grid. The tile size (64 symbols) is chosen to fit in the L1 cache of Apple M-series and Intel processors. Both row and column parities are updated in the same pass over each tile, avoiding the cache miss penalty of a separate second pass.

### Rayon parallelism

ECC blocks are independent: each block's parity is computed from its own symbols only. The `encode_parallel` and `decode_parallel` functions process each block on a separate Rayon thread, scaling linearly with the number of CPU cores.

---

## 10. Molecular Compiler: Rejection Sampling

### The synthesis constraint problem

DNA synthesis companies (Twist Bioscience, IDT) have constraints on the sequences they will synthesize. Sequences with extreme GC content (too high or too low) or long homopolymer runs (e.g., AAAAAAAA) are synthesis failures — they do not produce reliably uniform oligonucleotides. Additionally, certain sequence motifs are recognized as restriction enzyme sites and must be avoided.

The challenge: given arbitrary data to encode, how do you produce a DNA sequence that encodes that data AND satisfies all synthesis constraints?

### Rejection sampling with LCG seeds

Think of it like trying to write a poem that conveys a specific message while also conforming to a specific meter and rhyme scheme. If the first attempt does not work, you try again with a slightly different word order.

TOMB's molecular compiler uses **rejection sampling with a Linear Congruential Generator (LCG) seed**:

1. For each data payload, choose an LCG seed (starting from offset 0)
2. Use the LCG to generate a mapping from topological symbols to specific DNA bases
3. Apply the mapping to produce a candidate 196-nt strand
4. Check all synthesis constraints:
   - GC content between 40% and 65%
   - No homopolymer run longer than 4 consecutive identical bases
   - No forbidden restriction sites
   - Toehold accessible (positions 25–34 unpaired in secondary structure)
5. If the strand passes: record the seed offset and emit the strand
6. If the strand fails: increment the seed offset by 1 and try again (up to 229 times)

The seed offset is recorded in the FASTA header (`seed=<N>`) because the decoder needs to apply the same LCG mapping to recover the symbols from the DNA sequence.

### Why 229 tries?

At the default compiler settings, a random seed produces a valid strand approximately 70–80% of the time (given GC content in the 40–65% range and toehold accessibility requirements). With 229 tries, the probability of failing to find a valid seed for any single strand is astronomically small: (0.3)^229 ≈ 10^{-120}. In practice, the compiler achieves 100% valid strand rate across all tested inputs.

### span_fold: toehold accessibility validation

The most expensive constraint check is toehold accessibility: whether positions 25–34 are unpaired in the predicted minimum free energy (MFE) secondary structure of the full 196-nt strand.

The full ViennaRNA package (external tool, written in C) can compute this exactly, but calling it as a subprocess for every candidate strand across 1.6 million strands would take hours. TOMB's solution is `span_fold`: a pure Rust implementation of the McCaskill (1990) partition function, restricted to base pairs within ±80 nucleotides of the toehold.

The span restriction is justified: the kinetically relevant window for probe binding is 60–80 nt. Base pairs more than 80 nt away from the toehold do not affect toehold accessibility on the timescale of probe diffusion. The span restriction reduces the computation from O(n²) to O(n × L) where L = 80, running in approximately 0.5ms per strand.

`span_fold` is used as a **fast pre-filter**:
```
For each LCG seed attempt:
  → Fast checks (homopolymer, GC): O(n), ~1µs
  → span_fold accessibility: O(n×L), ~0.5ms
  → If passes: accept strand (no ViennaRNA subprocess needed)
  → If fails: try next seed
  → If all seeds exhausted: call ViennaRNA as final fallback
```

In practice, span_fold eliminates ViennaRNA subprocess calls for virtually all strands (span_fold has approximately 78% recall versus ViennaRNA on the validated test set). ViennaRNA is available as a final fallback when the fast filter exhausts all seeds.

### Thermodynamic parameters

span_fold uses Turner 2004 DNA stacking energy parameters at 37°C (physiological temperature), and hairpin/bulge/loop parameters from SantaLucia & Hicks 2004. These are the same parameters used by ViennaRNA 2.7.2.

The toehold accessibility criterion: positions 25–34 MUST appear as unpaired bases (dots, not brackets) in the dot-bracket notation of the predicted minimum free energy structure.

---

## 11. Phase Roadmap

Each phase adds a new layer to the protocol stack. Earlier phases are prerequisites for later phases.

### Phase 0 — Core Protocol (COMPLETE)

**What it adds**: The foundational encoding stack.

Components:
- `TopologicalSymbol`: 256-symbol knot-theory alphabet, 4× Shannon capacity
- `FractalDeltaEncoder` / `FractalDeltaDecoder`: two-stage compression
- `NucleotidePacker` / `NucleotideUnpacker`: FASTA-aware 2-bit packing
- `TopologicalECC`: √n × √n lattice parity, 12.5% overhead, all 4 DNA error types
- `HilbertAddress3D`: Skilling's 3D Hilbert curve, bijection verified
- `TombIndex`: named file storage, O(1) lookup, CRC32 integrity, 3 replicas
- `TOMBHardwareDriver` HAL: 4 frozen methods
- `SoftwareSimulatorDriver`: 4 hardware profiles (DNA, memristor, NVMe, degraded)
- `MolecularCompiler`: rejection sampling, span_fold, 196-nt strand assembly

**Status**: 1,396 tests passing, 0 failures. Benchmarked on 240 MB human chromosome 1.

### Phase 0.5 — Protocol Maturity (IN PROGRESS)

**What it adds**: Formal standardization infrastructure needed for SNIA submission.

Components (planned):
- 12-test formal conformance suite
- Plugin trait refactor: `TombCompressor`, `TombECC`, `TombHardware` as formal trait objects
- Multi-language SDK: Python, TypeScript, Go bindings
- Formal protocol specification document
- `TombFileDriver` and `TombRESTDriver` reference implementations

### Phase 1 — Standard Interface — tomb-tsi (COMPLETE)

**What it adds**: A uniform API that any software can use to access DNA storage without knowing about the encoding.

Components:
- REST API on port 3000 (Axum, async Tokio)
- MCP server on port 3001 (AI agent protocol)
- Endpoints: `/v1/health`, `/v1/version`, `/v1/compile`, `/v1/decode`, `/v1/verify-fasta`
- Shared application state (`AppState`) with pool directory management

### Phase 2 — Query Language — tomb-tql (COMPLETE)

**What it adds**: The ability to query data stored in DNA pools without decoding the entire pool.

Components:
- TQL lexer, parser, and AST
- Query planner with cost estimator and turnaround time model
- Semantic analyser
- Probe compiler: translates TQL queries into physical probe sequences
- Executor: runs queries against real or simulated pools

### Phase 3 — Molecular Compute — tomb-tmc (COMPLETE)

**What it adds**: The ability to run computations inside a DNA pool without moving data out.

Components:
- MISA (Molecular Instruction Set Architecture) instruction set
- Compute compiler: maps computation requests to MISA programs
- Strand designer: designs compute strands for specific operations
- Simulator: simulates MISA program execution in a DNA pool

### Phase 4 — Distributed Protocol — tomb-tdp (COMPLETE)

**What it adds**: Federated pools — multiple DNA pools operated by different parties, queried and written as one logical namespace.

Components:
- Federation manager: pool registry, health monitoring, role assignment
- Shard router: consistent hashing, range routing, geographic routing strategies
- Cross-pool executor: merges results from multiple pools
- Consensus monitor: detects divergence, generates repair plans

### Phase 5 — AI Integration — tomb-tai (COMPLETE)

**What it adds**: DNA as persistent memory for AI systems.

Components:
- Embedding store: store high-dimensional float vectors in DNA, search by cosine similarity
- Model archival: pack neural network weights as DNA strands with cost comparison
- DNA vector database: semantic search over DNA-stored documents

### Phase 6 — Secure Enclave — tomb-tse (COMPLETE)

**What it adds**: Cryptographic security native to molecular storage.

Components:
- Molecular OTP (One-Time Pad): key pool stored in DNA, physically irreversible consumption
- Toehold-gated access control: strands whose toehold only opens under the right probe key
- Copy-number integrity: auditing that detects unauthorized strand copying
- Security levels: Open, Restricted, Confidential, Secret

---

## 12. SNIA DARS Conformance

### The SNIA DNA Data Storage Alliance

The SNIA DNA Data Storage Alliance published its foundational whitepaper in June 2025, establishing the vocabulary and high-level architecture for standardized DNA storage systems. The whitepaper explicitly calls for "standard open-source codecs" to emerge as the foundation for interoperability.

TOMB is designed from the ground up to be that codec. The terminology throughout TOMB's codebase and documentation uses SNIA DARS terms (Pool, Strand, Codec, Index, Probe) to ensure direct conceptual alignment.

### Conformance layers

TOMB's SNIA conformance is defined at three levels:

**Codec conformance**: An implementation is codec-conformant if it can encode any byte sequence into TOMB strands and decode those strands back to the identical byte sequence. This is verified by the round-trip conformance tests.

**Protocol conformance**: An implementation is protocol-conformant if its strands have exactly the field structure specified in Section 6 of this document, use the specified Hilbert addressing, and encode protocol version in the specified positions. This is verified by the structural conformance tests.

**Interface conformance**: An implementation is interface-conformant if it implements the 4 HAL methods with the specified signatures and semantics. This is verified by the HAL conformance tests.

### Conformance test approach

Each conformance test is self-contained: given a specific input, it specifies the exact expected output (or verifies a specific invariant). No conformance test depends on external hardware, network connectivity, or environment-specific configuration.

The 17-test conformance suite (C-01 to C-17):
1. Round-trip: all 256 byte values encode and decode correctly
2. Strand length: every compiled strand is exactly 196 nucleotides
3. Field boundaries: primer, spacer, toehold, payload, primer are at specified positions
4. Hilbert bijection: no two strands in a pool share an address
5. ECC correction: single-symbol substitution errors are corrected
6. ECC detection: two-symbol errors are detected
7. Strand loss recovery: a missing strand is reconstructed from column parities
8. GC bounds: every strand GC content is in [40%, 65%]
9. Homopolymer: no strand contains a homopolymer run longer than 4 bases
10. Toehold accessibility: toehold positions (25–34) are unpaired in predicted structure
11. Version encoding: protocol version round-trips through positions 19–20 of left primer
12. Hilbert-toehold consistency: the toehold nucleotides decode to the correct Hilbert index

### MUST / SHOULD / MAY requirements (IETF language)

The following normative requirements apply to any implementation claiming TOMB conformance:

- A conformant encoder MUST produce strands of exactly 196 nucleotides.
- A conformant encoder MUST place the toehold at positions 25–34 (1-indexed).
- A conformant encoder MUST encode the Hilbert index in the toehold as specified.
- A conformant encoder MUST produce strands with GC content in [40%, 65%].
- A conformant encoder MUST NOT produce homopolymer runs longer than 4 bases.
- A conformant encoder MUST encode the protocol version in positions 19–20 of the left primer.
- A conformant decoder MUST correctly decode any strand produced by a conformant encoder at the same major version.
- A conformant decoder MUST correctly decode any strand produced by a conformant encoder at a lower minor version within the same major version.
- A conformant decoder SHOULD attempt to decode strands from a newer minor version.
- A conformant decoder SHOULD return a warning (not an error) for strands from a newer major version.
- A conformant ECC implementation MUST correct single-symbol errors.
- A conformant ECC implementation MUST detect double-symbol errors.
- A conformant HAL implementation MUST implement all 4 core methods.
- A conformant HAL implementation MAY implement the optional bulk_write, erase, and transaction methods.

---

## 13. Protocol Versioning

### The backward compatibility guarantee

A strand written today MUST be decodable by any TOMB decoder built in the future. This is the archival guarantee: data stored in DNA for 10,000 years must still be recoverable by technology that does not exist yet.

The protocol version field implements this guarantee. It is encoded in positions 19–20 of the left primer (0-indexed: bases at index 18 and 19 of the primer sequence).

### Version encoding scheme

Each position encodes one version component as a single nucleotide:

```
Position 19 (major version):   A=1, C=2, G=3, T=4
Position 20 (minor version):   A=0, C=1, G=2, T=3

Current version: v1.0 → "AA" in positions 19–20
```

Examples:
```
v1.0 → "AA"   (current)
v1.1 → "AC"
v1.2 → "AG"
v1.3 → "AT"
v2.0 → "CA"
v3.0 → "GA"
v4.3 → "TT"
```

This scheme supports 4 major versions and 4 minor versions (0–3) within a single-nucleotide encoding. When more than 4 minor versions are needed, TOMB will define an extension encoding scheme in a future specification version.

### Compatibility rules

These rules are implemented in `tomb/src/version.rs`:

- **FullyCompatible**: strand major = decoder major AND strand minor ≤ decoder minor. The decoder MUST handle this correctly.
- **MinorVersionNewer**: strand major = decoder major AND strand minor > decoder minor. The decoder SHOULD attempt to decode and return a warning about unknown features.
- **MajorVersionNewer**: strand major > decoder major. The decoder SHOULD attempt to decode and return a prominent warning. Breaking changes are permitted across major versions.

### What constitutes a major version bump

The following changes require a major version bump (potentially breaking existing decoders):
- Any change to the strand field boundaries (primer length, spacer length, toehold length, payload length)
- Any change to the Hilbert encoding of the toehold
- Any change to the topological symbol bijection (the byte-to-symbol mapping)
- Any change to the ECC lattice structure

The following changes are backwards-compatible (minor version bump):
- New optional header metadata fields
- New compression algorithms (old algorithms must still be supported)
- New HAL method implementations (additive only)
- Performance improvements that do not change output format

---

## 14. Cross-References

Each crate in the TOMB workspace has its own architecture documentation covering the internals specific to that layer:

- [tomb — Core Library Architecture](tomb/ARCHITECTURE.md)
- [tomb-tsi — Standard Interface Architecture](tomb-tsi/ARCHITECTURE.md)
- [tomb-tql — Query Language Architecture](tomb-tql/ARCHITECTURE.md)
- [tomb-tmc — Molecular Compute Architecture](tomb-tmc/ARCHITECTURE.md)
- [tomb-tdp — Distributed Protocol Architecture](tomb-tdp/ARCHITECTURE.md)
- [tomb-tai — AI Integration Architecture](tomb-tai/ARCHITECTURE.md)
- [tomb-tse — Secure Enclave Architecture](tomb-tse/ARCHITECTURE.md)

Root-level documents:
- [Root README](README.md) — user-facing overview and getting started guide
- [EXPLAINER.md](EXPLAINER.md) — extended conceptual explanation

Research and specification documents (in `tomb/docs/`):
- `tomb/docs/specs/tomb_spec.md` — formal protocol specification (in progress, Phase 0.5)
- `tomb/docs/specs/tomb_design_history.md` — all design decisions and their rationale
- `tomb/docs/molecular/design.md` — molecular compiler design decisions in detail
- `tomb/docs/research/context.md` — competitive landscape and prior art analysis
- `tomb/docs/research/papers_index.md` — 15 key research papers with notes

---

*TOMB — Topological Object Molecular Binary Storage*
*Patent Pending IN202641065626 | AGPL-3.0 | The TCP/IP of molecular storage*
