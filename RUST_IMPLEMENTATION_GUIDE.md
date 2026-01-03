# Rust JPEG 2000 Codec Implementation Guide

This document provides a detailed plan, architectural blueprint, and technical specifications for implementing a feature-complete JPEG 2000 codec in Rust, equivalent to OpenJPEG.

## 1. Introduction & Scope

The goal is to create a safe, idiomatic, and high-performance Rust implementation of the JPEG 2000 standard (ISO/IEC 15444-1) and supported extensions (ISO/IEC 15444-2) as found in OpenJPEG.

### Supported Features (Scope)
*   **Profiles**: Profile 0, 1 (standard), Cinema 2K/4K, IMF, Broadcast.
*   **Transforms**: 5-3 Reversible DWT (Integer), 9-7 Irreversible DWT (Floating point).
*   **Quantization**: Scalar quantization, derived and expounded.
*   **Entropy Coding**: Tier 1 EBCOT (Context-adaptive arithmetic coding).
*   **Codestream**: Tier 2 Packet coding, Tag Trees, Markers (SOC, SOT, SOD, EOC, SIZ, COD, QCD, PLT, TLM, PPM, PPT, etc.).
*   **Progression Orders**: LRCP, RLCP, RPCL, PCRL, CPRL.
*   **File Format**: JP2 (boxes: `jP`, `ftyp`, `jp2h`, `jp2c`, `colr`, `ihdr`, `pclr`, `cmap`, `cdef`).
*   **Color Spaces**: sRGB, Greyscale, YUV, e-YCC, CMYK.
*   **Advanced Features**: ROI (Region of Interest), Tile Parts, Partial Decoding, Multi-Component Transforms (MCT).

---

## 2. Architecture & Data Structures

The project should be organized as a Cargo workspace with core libraries and optional CLI tools.

### Workspace Structure
```
jpeg2000-rs/
├── Cargo.toml
├── jpeg2000-core/      (The main codec library)
│   ├── src/
│   │   ├── lib.rs
│   │   ├── image.rs    (Image data structures)
│   │   ├── tcd.rs      (Tile Coder/Decoder orchestration)
│   │   ├── t1.rs       (Tier 1 Entropy Coding)
│   │   ├── t2.rs       (Tier 2 Packet Coding)
│   │   ├── dwt.rs      (Discrete Wavelet Transform)
│   │   ├── mct.rs      (Multi-Component Transform)
│   │   ├── mqc.rs      (MQ Arithmetic Coder)
│   │   ├── bio.rs      (Bit I/O)
│   │   ├── cio.rs      (Stream I/O / Byte I/O)
│   │   ├── pi.rs       (Packet Iterator)
│   │   ├── tgt.rs      (Tag Tree)
│   │   └── event.rs    (Logging/Callbacks)
└── jpeg2000-cli/       (CLI binaries)
```

### Core Data Structures

#### Image & Component
```rust
pub struct Image {
    pub x0: u32, pub y0: u32, // Image offset
    pub x1: u32, pub y1: u32, // Image size
    pub numcomps: u32,
    pub color_space: ColorSpace,
    pub components: Vec<ImageComp>,
    pub icc_profile: Option<Vec<u8>>,
}

pub struct ImageComp {
    pub dx: u32, pub dy: u32, // Subsampling
    pub w: u32, pub h: u32,
    pub x0: u32, pub y0: u32,
    pub prec: u32,            // Bit depth
    pub sgnd: bool,           // Signedness
    pub alpha: u16,           // Alpha channel info
    pub data: Vec<i32>,       // Pixel data
}
```

#### Coding Parameters
```rust
pub struct CodingParameters {
    pub tile_size_on: bool,
    pub tdx: u32, pub tdy: u32, // Tile size
    pub tx0: u32, pub ty0: u32, // Tile offset
    pub irreversible: bool,     // 9-7 (true) or 5-3 (false)
    pub num_resolutions: u32,
    pub cblock_w_init: u32,
    pub cblock_h_init: u32,
    pub prog_order: ProgOrder,
    pub num_layers: u32,
    pub roi_compno: i32,        // ROI component index
    pub roi_shift: i32,         // ROI shift value
    pub use_jpwl: bool,         // Wireless extension support
    // ...
}
```

---

## 3. Core Math & Transform Specifications

### 3.1 Discrete Wavelet Transform (DWT)

#### 5-3 Reversible DWT (Integer)
Used for lossless compression. Implemented using lifting steps.
*   **Forward Transform**:
    $$Y(2n+1) = X(2n+1) - \lfloor \frac{X(2n) + X(2n+2)}{2} \rfloor$$
    $$Y(2n) = X(2n) + \lfloor \frac{Y(2n-1) + Y(2n+1) + 2}{4} \rfloor$$
*   **Inverse Transform**:
    $$X(2n) = Y(2n) - \lfloor \frac{Y(2n-1) + Y(2n+1) + 2}{4} \rfloor$$
    $$X(2n+1) = Y(2n+1) + \lfloor \frac{X(2n) + X(2n+2)}{2} \rfloor$$

**Boundary Handling**: Symmetric extension (mirroring).
Example: `X(-1) = X(1)`, `X(N) = X(N-2)`.

#### 9-7 Irreversible DWT (Floating Point)
Used for lossy compression. Implemented with 4 lifting steps + scaling.
Coefficients (standard values):
*   $\alpha = -1.586134342$
*   $\beta = -0.052980118$
*   $\gamma = 0.882911075$
*   $\delta = 0.443506852$
*   $K = 1.230174105$

**Forward Steps**:
1.  $X_{2n+1} \leftarrow X_{2n+1} + \alpha(X_{2n} + X_{2n+2})$
2.  $X_{2n} \leftarrow X_{2n} + \beta(X_{2n-1} + X_{2n+1})$
3.  $X_{2n+1} \leftarrow X_{2n+1} + \gamma(X_{2n} + X_{2n+2})$
4.  $X_{2n} \leftarrow X_{2n} + \delta(X_{2n-1} + X_{2n+1})$
5.  Scaling: High-pass samples $\times K$, Low-pass samples $\times (1/K)$

### 3.2 Multi-Component Transform (MCT)

#### Reversible Color Transform (RCT)
Used with 5-3 DWT.
*   **Forward**:
    $$Y_0 = \lfloor \frac{R + 2G + B}{4} \rfloor$$
    $$Y_1 = B - G$$
    $$Y_2 = R - G$$
*   **Inverse**:
    $$G = Y_0 - \lfloor \frac{Y_2 + Y_1}{4} \rfloor$$
    $$R = Y_2 + G$$
    $$B = Y_1 + G$$

#### Irreversible Color Transform (ICT)
Used with 9-7 DWT. Similar to YCbCr conversion using standard coefficients.

#### Custom MCT (Part 2 Extension)
OpenJPEG supports custom decorrelation matrices via `opj_mct_encode_custom` and `opj_mct_decode_custom`.
*   **Algorithm**: Matrix multiplication of component vector by a custom transformation matrix.
*   **Requirements**: Reading/Writing `MCC` (Multiple Component Collection) and `MCO` (Multiple Component Output) markers if encoding, or applying them if decoding.

---

## 4. Entropy Coding (Tier 1)

Tier 1 uses the **EBCOT** (Embedded Block Coding with Optimal Truncation) algorithm and the **MQ Coder** (Context-adaptive Arithmetic Coder).

### 4.1 MQ Coder
The MQ coder is a byte-oriented arithmetic coder.

#### State Tables
The state transition table `mqc_states` (47 states) defines probability estimation.
*   Structure: `{ qeval, mps, nmps, nlps }`
    *   `qeval`: Probability of LPS (Least Probable Symbol).
    *   `mps`: Most Probable Symbol (0 or 1).
    *   `nmps`: Next state index if MPS occurs.
    *   `nlps`: Next state index if LPS occurs.

**Table Snippet (First 5 states):**
| Index | Qe (Hex) | MPS | NMPS | NLPS |
| :--- | :--- | :--- | :--- | :--- |
| 0 | 0x5601 | 0 | 2 | 3 |
| 1 | 0x5601 | 1 | 3 | 2 |
| 2 | 0x3401 | 0 | 4 | 12 |
| 3 | 0x3401 | 1 | 5 | 13 |
| 4 | 0x1801 | 0 | 6 | 18 |

#### Implementation Caveats
*   **Byte Stuffing**: If a byte `0xFF` is written, the next byte must be `< 0x90`. The encoder inserts a stuffed bit if necessary.
*   **Renormalization**: When the interval `A` becomes too small, output bytes and double `A` and `C`.

### 4.2 EBCOT Context Modeling
Coding is done on **Code-Blocks** (typically 64x64 or 32x32) in 3 passes per bit-plane (from MSB to LSB).

#### The 3 Passes
1.  **Significance Propagation Pass (SPP)**: Encodes bits of samples that are currently insignificant but have significant neighbors.
2.  **Magnitude Refinement Pass (MRP)**: Encodes bits of samples that are already significant.
3.  **Cleanup Pass (CP)**: Encodes all remaining samples. Uses run-length coding for efficiency.

#### Context Labels (19 Contexts)
*   **Zero Coding (ZC)**: 9 contexts. Based on significance of 8 neighbors (H, V, D).
*   **Sign Coding (SC)**: 5 contexts. Based on sign and significance of H and V neighbors.
*   **Magnitude Refinement (MR)**: 3 contexts. Based on first refinement state.
*   **Run-Length (RL)**: 1 context (Uniform) + 1 context (Run-length).

**Zero Coding Lookup (ZC) Logic**:
Depends on the sub-band orientation (LL, HL, LH, HH). The `lut_ctxno_zc` table maps the neighbor configuration pattern to a context index (0-8).

---

## 5. Codestream Syntax & Tier 2

Tier 2 organizes the compressed code-block data into **Packets**.

### 5.1 Tag Trees (TGT)
Used to encode:
1.  **Inclusion Information**: Which code-blocks are included in the packet.
2.  **Zero-bitplane Count**: Number of zero bit-planes for each code-block.

**Algorithm**:
*   A quad-tree structure where each node holds the minimum value of its children.
*   Encoding: Send the difference between the node value and the current threshold, moving up from leaf to root.

### 5.2 Packet Headers
A packet header contains:
1.  **Zero-bitplane inclusion** (via Tag Tree).
2.  **Number of coding passes** (variable length coding).
3.  **Length of compressed data** (variable length coding).

**Bit I/O (BIO)**: Packet headers are written bit-by-bit using a Bit Writer.

### 5.3 Markers
*   `SOC` (0xFF4F): Start of Codestream.
*   `SOT` (0xFF90): Start of Tile.
*   `SOD` (0xFF93): Start of Data.
*   `EOC` (0xFFD9): End of Codestream.
*   `SIZ` (0xFF51): Image and tile size.
*   `COD` (0xFF52): Coding style defaults.
*   `QCD` (0xFF5C): Quantization default.
*   **Advanced Markers**: `TLM` (Tile-part lengths), `PLT` (Packet length, tile-part), `PPM`/`PPT` (Packet headers).

---

## 6. Advanced Features (OpenJPEG Parity)

### 6.1 Region of Interest (ROI)
OpenJPEG implements the **Maxshift** method.
*   **Encoding**: Coefficients belonging to the ROI are shifted up by `s` bits, where `s` is sufficient to ensure ROI coefficients are larger than any non-ROI coefficient.
*   **Decoding**: Any coefficient magnitude $M \ge 2^s$ belongs to the ROI. The coefficient is shifted down by `s` bits.
*   **Formula**: $s = \max(M_{non-ROI})$.

### 6.2 JP2 Metadata Boxes
To ensure full capability match, the codec must parse/write:
*   **Palette (pclr)**: Maps single-channel values to multi-channel colors (LUT).
*   **Component Mapping (cmap)**: Maps components to palette columns or direct channels.
*   **Channel Definition (cdef)**: Defines which channel is Alpha, Red, Green, Blue, etc.

### 6.3 Tile Parts
A tile can be split into multiple **Tile Parts** (`SOT`...`SOD`...`SOT`...).
*   **Implementation**: The decoder must accumulate packets from all tile-parts for a given tile index before decoding the tile.
*   **Indexing**: `TLM` markers speed up seeking to specific tile-parts.

### 6.4 Partial Decoding
OpenJPEG supports decoding:
*   **Reduced Resolution**: Discarding higher DWT levels (resolution reduction).
*   **Area of Interest**: Decoding only code-blocks that intersect a requested window.
*   **Layer Decoding**: Decoding only the first N quality layers.

---

## 7. Implementation Roadmap

1.  **Foundation**:
    *   Implement `BitReader`/`BitWriter` (BIO).
    *   Implement `ByteReader`/`ByteWriter` (CIO) with Big Endian support.
    *   Define `Image` and `CodingParameters` structs.

2.  **Codestream Parsing**:
    *   Implement marker parsing (`SIZ`, `COD`, `QCD`, `TLM`, `PLT`).
    *   Implement `Tile` and `Component` setup logic.

3.  **Tier 1 (Entropy)**:
    *   Implement `MqCoder` (Enc/Dec) with full state table.
    *   Implement `T1Coder` with the 3 EBCOT passes and context lookup tables.
    *   **Verify**: Test against known bit-patterns (e.g., from OpenJPEG tests).

4.  **Math & Transform**:
    *   Implement `Dwt` (Forward/Inverse, 5-3/9-7).
    *   Implement `Mct` (RCT/ICT and Custom Matrix).
    *   Implement `Quantization` and `ROI` scaling.

5.  **Tier 2 (Packets)**:
    *   Implement `TagTree`.
    *   Implement `PacketIterator` (progression order logic).
    *   Implement `T2Coder` (Packet Header reading/writing).

6.  **TCD Orchestration**:
    *   Connect T2 -> T1 -> DWT -> MCT for decoding.
    *   Connect MCT -> DWT -> T1 -> T2 for encoding.
    *   Implement Partial Decoding logic (skipping resolutions/layers).

7.  **JP2 File Format**:
    *   Wrap codestream in JP2 boxes (`ftyp`, `jp2h`, `jp2c`).
    *   Implement `pclr`, `cmap`, `cdef` handling.

---

## 8. Caveats & Pitfalls

1.  **Arithmetic Coder (MQC) Adaptation**:
    *   The probability estimation state machine is fragile. A single bit error desynchronizes the decoder. Ensure the `switch` behavior (LPS/MPS exchange) exactly matches the standard.
    *   **Byte Stuffing**: Watch out for the `0xFF` byte logic. If `0xFF` is generated, the carry bit logic in the encoder and the bit reading logic in the decoder must handle it specifically (skipping the next bit or byte).

2.  **Floating Point Determinism**:
    *   The 9-7 DWT uses floats. Different architectures/compilers might produce slightly different results. For strict conformance, ensure consistent rounding modes or use fixed-point approximations where appropriate (though OpenJPEG uses `float`/`double`).

3.  **Boundary Conditions**:
    *   DWT Symmetric Extension: Incorrect mirroring at image boundaries is a common source of artifacts (e.g., ringing at edges).
    *   Code-block boundaries: Context modeling depends on neighbors. Neighbors outside the code-block are considered 0 (non-significant).

4.  **Endianness**:
    *   JPEG 2000 is **Big Endian**. Rust is typically Little Endian on x86/ARM. Always use `u32::to_be()` / `u32::from_be()` or helper functions in `cio` module.

5.  **Performance**:
    *   Tier 1 (Entropy Coding) is the bottleneck (bit-level sequential processing).
    *   **Rust Tip**: Use `rayon` to parallelize code-block decoding. Since code-blocks are independent, this scales perfectly.

6.  **Security**:
    *   Marker segments have length fields. **Always** validate that `length` does not exceed the remaining buffer size to avoid OOB reads/panics.
    *   Tag Tree recursion: Limit depth or validate sizes to prevent stack overflow on malicious inputs.
