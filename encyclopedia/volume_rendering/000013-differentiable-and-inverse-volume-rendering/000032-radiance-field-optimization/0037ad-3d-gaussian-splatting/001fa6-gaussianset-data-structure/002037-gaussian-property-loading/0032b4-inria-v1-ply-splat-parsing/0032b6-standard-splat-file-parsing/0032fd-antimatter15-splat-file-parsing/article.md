# Antimatter15 .splat File Parsing

Also called: Antimatter15 .splat file reader, Antimatter15 Splat Reader, Lazy ChunkSource for .splat

## Overview
This method implements a lazy `ChunkSource` for reading Antimatter15 `.splat` files, which are flat binary arrays of 32-byte records with no header [S1]. It decodes position, geometric (rotation, scale, opacity), and DC-only color layers on demand, supporting both contiguous chunk reads and scatter-gather reads [S1]. Unlike formats with spherical harmonic coefficients, `.splat` files carry no SH data, so the color layer is DC-only (constant per gaussian) [S1].

## How it works
**Inputs and Outputs:**
- Input: A `.splat` file path or buffer.
- Output: A `ChunkSource` object providing lazy access to splat properties.

**Data Structures:**
- **Record Layout (32 bytes per splat):**
  - `position`: 3 × `Float32` at offsets 0, 4, 8.
  - `scale`: 3 × `Float32` at offsets 12, 16, 20.
  - `color`: 4 × `Uint8` at offsets 24, 25, 26, 27.
  - `rotation`: 4 × `Uint8` at offsets 28, 29, 30, 31.
- **Constants:**
  - `BYTES_PER_SPLAT = 32` [S1].
  - `SH_C0 = 0.28209479177387814` [S1].
  - `eps = 1e-6` [S1].

**Steps:**
1. **Initialization:**
   - Determine file size. If undefined, read all data into a resident buffer; otherwise use `source.size` [S1].
   - Validate that `size` is a multiple of 32 bytes and non-zero [S1].
   - Calculate `numGaussians = size / 32` [S1].
   - Calculate `chunkSize` from `pool.chunkSize` [S1].
   - Calculate `numChunks = max(1, ceil(numGaussians / chunkSize))` [S1].
   - Define available layers as a set containing `'position'`, `'geometric'`, `'color'` [S1].
   - Define layouts:
     - `position`: stride `POSITION_STRIDE`, fields `positionFields()` [S1].
     - `geometric`: stride `GEOMETRIC_STRIDE`, fields `geometricFields()` [S1].
     - `color`: stride `colorStride(0)`, fields `colorFields(0)` [S1].

2. **Read Request Handling:**
   - If `lod != 0`, throw an error [S1].
   - Extract `position`, `geometric`, `color` `Float32Arrays` from the request [S1].
   - If no layers are requested, return [S1].

3. **Gather Read Path (Scatter-Gather):**
   - If the request has indices, sort slots by source row to coalesce reads [S1].
   - Iterate over `gatherRuns`, calculating `firstByte` and `byteLen` [S1].
   - If the data is resident, use `subarray`; otherwise allocate/reuse `gatherScratch` and call `readExact` [S1].
   - Call `decode` with `srcRec` mapping [S1].

4. **Contiguous Chunk Decode Loop:**
   - Iterate `t` from 0 to `n-1` [S1].
   - Calculate byte offset `o = srcRec(t) * 32` [S1].
   - Calculate destination slot `d = dstRow(t)` [S1].
   - Call `decodePosition`, `decodeGeometric`, `decodeColor` if respective layers are requested [S1].

**Decoding Formulas:**
- **Position:** Read 3 floats from `DataView` at offset `o` with little-endian byte order [S1].
- **Rotation:**
  - Read 4 `Uint8`s at offsets `o+28` to `o+31` [S1].
  - Map to `[-1, 1]` via `(u8 / 255) * 2 - 1` [S1].
  - Normalize quaternion. If length is 0, set to identity `(1, 0, 0, 0)` [S1].
- **Scale:**
  - Read 3 floats from `DataView` at offsets `o+12`, `o+16`, `o+20` [S1].
  - Apply `log` [S1].
- **Opacity:**
  - Read `Uint8` at `o+27` [S1].
  - Clamp normalized alpha to `[eps, 1-eps]` where `eps = 1e-6` [S1].
  - Apply inverse sigmoid (logit): `log(a / (1 - a))` [S1].
- **Color:**
  - Read 3 `Uint8`s at offsets `o+24`, `o+25`, `o+26` [S1].
  - Map to `[-0.5, 0.5]` by `(u8 / 255 - 0.5)` [S1].
  - Divide by `SH_C0` [S1].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import math

BYTES_PER_SPLAT = 32
SH_C0 = 0.28209479177387814
EPS = 1e-6

def decode_antimatter_splat(file_data, num_gaussians, chunk_size, request_layers):
    """
    Lazy ChunkSource for Antimatter15 .splat files.
    file_data: bytes or memory view of the file content.
    num_gaussians: int, total number of splats.
    chunk_size: int, size of each chunk in splats.
    request_layers: set of strings, e.g., {'position', 'geometric', 'color'}.
    """
    if len(file_data) % BYTES_PER_SPLAT != 0:
        raise ValueError("File size must be a multiple of 32 bytes")
    
    num_chunks = max(1, math.ceil(num_gaussians / chunk_size))
    
    # Pre-calculate strides and field counts if needed by the specific ChunkSource implementation
    # For this generic parser, we focus on the decode logic.
    
    def decode_position(src_rec_idx, dst_array, dst_offset):
        """Decode position from a single record."""
        base = src_rec_idx * BYTES_PER_SPLAT
        # Read 3 floats little-endian
        x = struct.unpack_from('<f', file_data, base + 0)
        y = struct.unpack_from('<f', file_data, base + 4)
        z = struct.unpack_from('<f', file_data, base + 8)
        dst_array[dst_offset] = x
        dst_array[dst_offset + 1] = y
        dst_array[dst_offset + 2] = z

    def decode_geometric(src_rec_idx, dst_array, dst_offset):
        """Decode scale and rotation from a single record."""
        base = src_rec_idx * BYTES_PER_SPLAT
        
        # Scale: 3 floats at 12, 16, 20
        sx = struct.unpack_from('<f', file_data, base + 12)
        sy = struct.unpack_from('<f', file_data, base + 16)
        sz = struct.unpack_from('<f', file_data, base + 20)
        # Apply log
        dst_array[dst_offset] = math.log(sx)
        dst_array[dst_offset + 1] = math.log(sy)
        dst_array[dst_offset + 2] = math.log(sz)
        
        # Opacity: 1 uint8 at 27
        alpha_u8 = struct.unpack_from('B', file_data, base + 27)[0]
        alpha_norm = alpha_u8 / 255.0
        # Clamp to [eps, 1-eps]
        alpha_clamped = max(EPS, min(1.0 - EPS, alpha_norm))
        # Inverse sigmoid (logit)
        opacity = math.log(alpha_clamped / (1.0 - alpha_clamped))
        dst_array[dst_offset + 3] = opacity
        
        # Rotation: 4 uint8 at 28, 29, 30, 31
        r0_u8 = struct.unpack_from('B', file_data, base + 28)[0]
        r1_u8 = struct.unpack_from('B', file_data, base + 29)[0]
        r2_u8 = struct.unpack_from('B', file_data, base + 30)[0]
        r3_u8 = struct.unpack_from('B', file_data, base + 31)[0]
        
        # Map to [-1, 1]
        q0 = (r0_u8 / 255.0) * 2.0 - 1.0
        q1 = (r1_u8 / 255.0) * 2.0 - 1.0
        q2 = (r2_u8 / 255.0) * 2.0 - 1.0
        q3 = (r3_u8 / 255.0) * 2.0 - 1.0
        
        # Normalize
        norm = math.sqrt(q0*q0 + q1*q1 + q2*q2 + q3*q3)
        if norm == 0:
            # Identity quaternion
            q0, q1, q2, q3 = 1.0, 0.0, 0.0, 0.0
        else:
            q0 /= norm
            q1 /= norm
            q2 /= norm
            q3 /= norm
            
        dst_array[dst_offset + 4] = q0
        dst_array[dst_offset + 5] = q1
        dst_array[dst_offset + 6] = q2
        dst_array[dst_offset + 7] = q3

    def decode_color(src_rec_idx, dst_array, dst_offset):
        """Decode DC-only color from a single record."""
        base = src_rec_idx * BYTES_PER_SPLAT
        r_u8 = struct.unpack_from('B', file_data, base + 24)[0]
        g_u8 = struct.unpack_from('B', file_data, base + 25)[0]
        b_u8 = struct.unpack_from('B', file_data, base + 26)[0]
        
        # Map to [-0.5, 0.5]
        r = (r_u8 / 255.0) - 0.5
        g = (g_u8 / 255.0) - 0.5
        b = (b_u8 / 255.0) - 0.5
        
        # Divide by SH_C0
        dst_array[dst_offset] = r / SH_C0
        dst_array[dst_offset + 1] = g / SH_C0
        dst_array[dst_offset + 2] = b / SH_C0

    def read_chunk(chunk_idx, layers):
        """Read a specific chunk of splats."""
        start_idx = chunk_idx * chunk_size
        end_idx = min(start_idx + chunk_size, num_gaussians)
        count = end_idx - start_idx
        
        # Allocate output arrays based on requested layers
        # Note: In a real implementation, these would be pre-allocated buffers
        # passed in by the ChunkSource interface. Here we simulate the decode loop.
        
        for t in range(count):
            src_rec_idx = start_idx + t
            dst_offset = t * 8 # Assuming 8 floats per splat in output buffer for simplicity
            
            if 'position' in layers:
                decode_position(src_rec_idx, dst_array, dst_offset)
                dst_offset += 3
                
            if 'geometric' in layers:
                decode_geometric(src_rec_idx, dst_array, dst_offset)
                dst_offset += 4 # scale(3) + opacity(1) + rotation(4) -> wait, geometric usually includes scale, opacity, rotation. 
                # Based on notes: geometric fields include scale, opacity, rotation.
                # Let's assume the output buffer layout matches the source fields.
                # The pseudocode above decodes them into separate slots.
                # For the sake of the loop structure in [S1]:
                # dstRow(t) is the destination slot.
                pass 
            
            if 'color' in layers:
                decode_color(src_rec_idx, dst_array, dst_offset)
                dst_offset += 3

    return read_chunk
```

## Subtopics

### [Splatv Binary File Format Parser](7762-splatv-binary-file-format-parser.md)
Parses a custom binary file format (.splatv) containing magic header, JSON camera metadata, and binary splat data chunks.

## Operating characteristics
- **Time Complexity:** O(N) for reading N splats, with overhead for lazy decoding and scatter-gather coalescing.
- **Memory Complexity:** O(1) auxiliary memory per chunk, plus resident buffer if file size is undefined or small [S1].
- **Strengths:** Lazy loading allows efficient access to subsets of splats; scatter-gather optimization reduces I/O calls for non-contiguous access [S1].
- **Limitations:** No spherical harmonic data, limiting color fidelity to DC-only [S1]. Degenerate quaternions and opacity boundaries require explicit handling [S1].

## Background
The Antimatter15 `.splat` format is a compact binary representation for 3D Gaussian Splatting data, used in tools like `splat-transform` [S1]. It is designed for efficient storage and parsing of Gaussian properties without the overhead of spherical harmonic coefficients [S1].

## Sources
[S1] playcanvas/splat-transform, https://github.com/playcanvas/splat-transform, fetched 2026-10-27
