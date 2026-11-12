# Antimatter15 .splat File Parsing

Also called: Antimatter15 .splat file reader, Antimatter15 Splat Reader, Lazy ChunkSource for .splat

## Overview
This method implements a lazy `ChunkSource` for reading Antimatter15 `.splat` files, which are flat arrays of 32-byte binary records with no header [S1]. It decodes position, geometric (rotation, scale, opacity), and DC-only color layers on demand, supporting both contiguous chunk reads and scatter-gather reads [S1]. Unlike formats with spherical harmonic coefficients, `.splat` files carry no SH data, so the color layer is DC-only (constant per gaussian) [S1]. It builds on [Standard Splat File Parsing](6828-standard-splat-file-parsing.md) by providing a lazy, chunk-based interface rather than a simple bulk load.

## How it works
**Inputs and Outputs:**
- Input: A file path or byte stream representing an Antimatter15 `.splat` file.
- Output: A `ChunkSource` object that provides decoded Gaussian properties (position, geometric, color) via `read()` requests.

**Data Structures:**
- **Record Layout (32 bytes):**
  - `position`: 3 × `Float32` at offsets 0, 4, 8.
  - `scale`: 3 × `Float32` at offsets 12, 16, 20.
  - `color`: 4 × `uint8` at offsets 24, 25, 26, 27.
  - `rotation`: 4 × `uint8` at offsets 28, 29, 30, 31.
- **Constants:**
  - `BYTES_PER_SPLAT = 32` [S1].
  - `SH_C0 = 0.28209479177387814` [S1].
  - `EPSILON = 1e-6` [S1].

**Steps:**
1. **Initialization:**
   - Check file size. If size is undefined, read all data into a resident buffer; otherwise use `source.size` [S1].
   - Validate that `size` is a multiple of 32 bytes and non-zero [S1].
   - Calculate `numGaussians = size / 32` [S1].
   - Calculate `chunkSize` from `pool.chunkSize`. Calculate `numChunks = max(1, ceil(numGaussians / chunkSize))` [S1].
   - Define available layers as a Set containing `'position'`, `'geometric'`, `'color'` [S1].
   - Define layouts:
     - `position`: stride `POSITION_STRIDE`, fields `positionFields()` [S1].
     - `geometric`: stride `GEOMETRIC_STRIDE`, fields `geometricFields()` [S1].
     - `color`: stride `colorStride(0)`, fields `colorFields(0)` [S1].

2. **Read Request Handling:**
   - On `read` request: if `lod != 0`, throw error [S1].
   - Extract `position`, `geometric`, `color` `Float32Arrays` from request. If none requested, return [S1].
   - **Gather Read Path:** If request has indices, sort slots by source row to coalesce reads [S1].
   - Iterate over `gatherRuns`, calculating `firstByte` and `byteLen` [S1].
   - If resident, use `subarray`; otherwise allocate/reuse `gatherScratch` and call `readExact` [S1].
   - Call `decode` with `srcRec` mapping [S1].

3. **Decoding (Contiguous Chunk Loop):**
   - Iterate `t` from 0 to `n-1` [S1].
   - Calculate byte offset `o = srcRec(t) * 32` [S1].
   - Calculate destination slot `d = dstRow(t)` [S1].
   - Call `decodePosition`, `decodeGeometric`, `decodeColor` if respective layers are requested [S1].

**Decoding Formulas:**
- **Position:** Read 3 floats from `DataView` at offset `o` with little-endian byte order [S1].
- **Rotation:**
  - Read 4 `uint8`s at offsets `o+28` to `o+31` [S1].
  - Map to `[-1, 1]` via $q_i = \frac{u8_i}{255} \times 2 - 1$ [S1].
  - Normalize quaternion: $q_{norm} = \frac{q}{\|q\|}$ [S1].
  - If $\|q\| = 0$, set to identity $(1, 0, 0, 0)$ [S1].
- **Scale:**
  - Read 3 floats from `DataView` at offsets `o+12`, `o+16`, `o+20` [S1].
  - Apply log: $s_i = \log(scale_i)$ [S1].
- **Opacity:**
  - Read `uint8` at `o+27` [S1].
  - Normalize: $\alpha_{norm} = \frac{u8}{255}$ [S1].
  - Clamp to $[\epsilon, 1-\epsilon]$ where $\epsilon = 10^{-6}$ [S1].
  - Apply inverse sigmoid (logit): $\alpha = \log\left(\frac{\alpha_{clamp}}{1 - \alpha_{clamp}}\right)$ [S1].
- **Color:**
  - Read 3 `uint8`s at offsets `o+24`, `o+25`, `o+26` [S1].
  - Map to $[-0.5, 0.5]$ via $c_i = \frac{u8_i}{255} - 0.5$ [S1].
  - Divide by `SH_C0`: $C_i = \frac{c_i}{SH\_C0}$ [S1].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import math
import struct

BYTES_PER_SPLAT = 32
SH_C0 = 0.28209479177387814
EPSILON = 1e-6

def decode_splat_record(data: bytes, offset: int) -> dict:
    """Decode a single 32-byte splat record."""
    # Position: 3 floats at 0, 4, 8
    x, y, z = struct.unpack_from('<fff', data, offset)
    
    # Scale: 3 floats at 12, 16, 20
    sx, sy, sz = struct.unpack_from('<fff', data, offset + 12)
    # Apply log
    sx = math.log(sx)
    sy = math.log(sy)
    sz = math.log(sz)
    
    # Opacity: 1 uint8 at 27
    alpha_u8 = data[offset + 27]
    alpha_norm = alpha_u8 / 255.0
    # Clamp to [eps, 1-eps]
    alpha_clamped = max(EPSILON, min(1.0 - EPSILON, alpha_norm))
    # Inverse sigmoid
    opacity = math.log(alpha_clamped / (1.0 - alpha_clamped))
    
    # Rotation: 4 uint8 at 28, 29, 30, 31
    qx_u8, qy_u8, qz_u8, qw_u8 = data[offset + 28], data[offset + 29], data[offset + 30], data[offset + 31]
    # Map to [-1, 1]
    qx = (qx_u8 / 255.0) * 2.0 - 1.0
    qy = (qy_u8 / 255.0) * 2.0 - 1.0
    qz = (qz_u8 / 255.0) * 2.0 - 1.0
    qw = (qw_u8 / 255.0) * 2.0 - 1.0
    # Normalize
    norm = math.sqrt(qx**2 + qy**2 + qz**2 + qw**2)
    if norm == 0:
        qx, qy, qz, qw = 1.0, 0.0, 0.0, 0.0
    else:
        qx /= norm
        qy /= norm
        qz /= norm
        qw /= norm
        
    # Color: 3 uint8 at 24, 25, 26
    r_u8, g_u8, b_u8 = data[offset + 24], data[offset + 25], data[offset + 26]
    # Map to [-0.5, 0.5] and divide by SH_C0
    r = ((r_u8 / 255.0) - 0.5) / SH_C0
    g = ((g_u8 / 255.0) - 0.5) / SH_C0
    b = ((b_u8 / 255.0) - 0.5) / SH_C0
    
    return {
        'position': (x, y, z),
        'scale': (sx, sy, sz),
        'opacity': opacity,
        'rotation': (qx, qy, qz, qw),
        'color': (r, g, b)
    }

def parse_splat_file(file_path: str, chunk_size: int = 1000):
    """
    Lazy ChunkSource implementation for Antimatter15 .splat files.
    """
    with open(file_path, 'rb') as f:
        file_data = f.read()
    
    size = len(file_data)
    if size == 0 or size % BYTES_PER_SPLAT != 0:
        raise ValueError("Invalid file size")
    
    num_gaussians = size // BYTES_PER_SPLAT
    num_chunks = max(1, math.ceil(num_gaussians / chunk_size))
    
    class SplatChunkSource:
        def __init__(self):
            self.resident_data = file_data
            self.num_chunks = num_chunks
            self.chunk_size = chunk_size
            self.num_gaussians = num_gaussians
            
        def read(self, chunk_idx: int, indices: list = None):
            """
            Read a chunk of splats.
            If indices is provided, perform scatter-gather read.
            """
            if indices is None:
                # Contiguous read
                start = chunk_idx * self.chunk_size
                end = min(start + self.chunk_size, self.num_gaussians)
                splats = []
                for i in range(start, end):
                    offset = i * BYTES_PER_SPLAT
                    splats.append(decode_splat_record(self.resident_data, offset))
                return splats
            else:
                # Scatter-gather read
                # Sort indices to coalesce reads (simplified: just decode each)
                splats = []
                for idx in indices:
                    if idx < self.num_gaussians:
                        offset = idx * BYTES_PER_SPLAT
                        splats.append(decode_splat_record(self.resident_data, offset))
                    else:
                        raise IndexError(f"Index {idx} out of bounds")
                return splats

    return SplatChunkSource()
```

## Subtopics

### [Splatv Binary File Format Parser](7762-splatv-binary-file-format-parser.md)
The Splatv Binary File Format Parser is a method for loading `.splatv` files, a custom binary container format used by `gsplat.js` to store 3D Gaussian Splatting scene data. It solves the problem of parsing a structured binary stream that combines a magic header, JSON-encoded camera metadata, and binary splat data chunks into a usable scene graph. It is distinct from the Antimatter15 `.splat` parser because it handles a multi-chunk container with explicit type markers and embedded JSON metadata, rather than a flat, headerless array of records.

## Operating characteristics
- **Time Complexity:** $O(N)$ for full load, where $N$ is the number of Gaussians. Lazy loading reduces initial load time to $O(1)$ per chunk accessed.
- **Memory Complexity:** $O(N)$ for resident buffer, or $O(C)$ for chunk-based lazy loading, where $C$ is the chunk size.
- **Performance:** Depends on hardware and resolution. Lazy loading is beneficial for large scenes where not all Gaussians are needed immediately.
- **Strengths:** Low memory footprint for large scenes, fast initial load, supports scatter-gather reads for efficient random access.
- **Limitations:** No spherical harmonic data, so color is DC-only. Requires full file read for non-resident modes if random access is frequent.
- **When to use:** When memory is constrained or when only a subset of Gaussians is needed.
- **When not to use:** When full scene access is required and memory is sufficient, or when SH coefficients are needed.

## Background
The Antimatter15 `.splat` format is a simple, headerless binary format for storing 3D Gaussian Splatting data. It was popularized by the Antimatter15 project and is widely used in web-based 3D Gaussian Splatting viewers due to its simplicity and efficiency. The format stores position, scale, rotation, opacity, and color in a fixed 32-byte record.

## Sources
[S1] playcanvas/splat-transform, https://github.com/playcanvas/splat-transform, fetched 2026-10-27
