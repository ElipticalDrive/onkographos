# RLE4 Compression

Also called: Map4 generation, RLE4, RLE4 Voxel Compression, RLE4::compress, Slab-based RLE voxel encoding

## Overview
RLE4 Compression converts a dense 3D voxel grid into a sparse, column-oriented structure called **Map4** to solve memory bloat in large volumes with significant empty space [S1, unofficial]. It processes the volume slice-by-slice (XZ planes), encoding runs of empty space (skip) and occupied space (solid) along the Y-axis into 16-bit words, while storing per-voxel color and intensity data in a separate texture buffer [S1, unofficial]. Unlike [Binary Voxel RLE Encoding](1071-binary-voxel-rle-encoding.md) which stores binary occupancy, RLE4 preserves per-voxel material and intensity information for splatting [S1, unofficial]. It differs from [Run-length encoded voxel runs](1209-run-length-encoded-voxel-runs.md) by using a fixed 16-bit RLE format with specific bit-widths for solid (6 bits) and skip (10 bits) runs, and by including a perturbation step for surface normals [S1, unofficial].

## How it works
**Inputs and Outputs:**
- **Inputs:** `mem` (bit-packed 3D voxel grid), `col1` and `col2` (optional bit-packed material arrays), `sx`, `sy`, `sz` (volume dimensions), `mip_lvl` (mipmap level for scale calculation) [S1, unofficial].
- **Outputs:** `map4` structure containing `sx`, `sy`, `sz`, `map` (`uint*` array of size $sx \cdot sz$ storing slab offsets), `slabs` (`ushort*` array of concatenated slab data), and `slabs_size` (total byte size of `slabs`) [S1, unofficial].

**Data Structures:**
1. **Map4 Header**: `map[x + z*sx]` stores the index into the `slabs` array for each column $(x,z)$ [S1, unofficial].
2. **Slab Layout** (per column $x,z$): `[slab_len: ushort][tex_len: ushort][RLE entries: ushort...][Texture entries: ushort...]` [S1, unofficial]. `slab_len` is the count of RLE entries. `tex_len` is the count of texture entries [S1, unofficial].
3. **RLE Entry** (`ushort`, 16 bits): Bits 0-9 (10 bits) store skip run length ($0 \dots 1023$) [S1, unofficial]. Bits 10-15 (6 bits) store solid run length ($0 \dots 63$) [S1, unofficial]. Format: `(solid << 10) | skip`. If `solid==0`, it is a skip run. If `skip==0`, it is a solid run [S1, unofficial].
4. **Texture Entry** (`ushort`, 16 bits): Bits 0-7 (8 bits) store intensity [S1, unofficial]. Bits 8-9 (2 bits) store color index ($0 \dots 3$) [S1, unofficial]. Bits 10-15 are unused/padding. Packed as: `intensity | (color << 8)` [S1, unofficial].

**Numbered Steps:**
1. Initialize `map` array of size $sx \cdot sz$ [S1, unofficial].
2. Iterate $i$ from $0$ to $sx-1$ and $k$ from $0$ to $sz-1$ (X and Z coordinates):
   a. Record current slab offset: `map[i + k*sx] = slabs.size()` [S1, unofficial].
   b. Reserve space for headers: `slab_len_pos = slabs.size()`; `slabs.push_back(0)`; `tex_len_pos = slabs.size()`; `slabs.push_back(0)` [S1, unofficial].
   c. Initialize counters: `solid = 0`, `skip = 0`, `store = false` [S1, unofficial].
   d. Clear `texture` buffer [S1, unofficial].
   e. Iterate $j$ from $0$ to $sy$ (inclusive, to flush final run) [S1, unofficial]:
      i. Check if voxel $(i, j, k)$ is set in `mem` [S1, unofficial]. If $j < sy$: `f = (mem[(i + j*sx + k*sxy) >> 3] >> (i & 7)) & 1` [S1, unofficial]. If $j == sy$: `f = 0` [S1, unofficial].
      ii. If `f` is true (voxel is solid):
          - Compute perturbed normal: Initialize `mx=0, my=0, mz=0, cnt=0` [S1, unofficial]. Loop $a, b, c$ from $-1$ to $1$. If neighbor $(i+a, j+b, k+c)$ is within bounds and set in `mem`: `mx += a`, `my += b`, `mz += c`, `cnt++` [S1, unofficial].
          - If `cnt < 27` (not fully surrounded): `centroid = (-mx, my, -mz)` [S1, unofficial]. Normalize `centroid`. Quantize to 8-bit: `nx = clamp(int(127.0 * centroid.x + 128.0), 0, 255)`, same for `ny, nz` [S1, unofficial].
      iii. Extract material bits: `bit1 = (col1[...] & (1<<(i&7))) ? 256 : 0`. `bit2 = (col2[...] & (1<<(i&7))) ? 512 : 0`. `mat = (bit1 | bit2) >> 8` [S1, unofficial].
      iv. Calculate noise: `rnd1, rnd2, rnd3` using sine/cosine of coordinates scaled by `scale` and dimensions. `rnd4 = rnd1 + rnd2 + rnd3 + 3` [S1, unofficial].
      v. Perturb `ny` based on `mat`: If `mat == 0`: `ny = (ny * (((((i*scale) ^ (k*scale)) >> 4) & 15) + 15)) >> 5` [S1, unofficial]. If `mat == 3`: `ny = (ny * (((((i*scale) ^ (k*scale)) >> 3) & 15) + 25)) >> 5` [S1, unofficial]. Clamp `ny` to $0 \dots 255$ [S1, unofficial].
      vi. Store in texture: `texture.push_back(ny | bit1 | bit2)` [S1, unofficial]. Set `store = true` [S1, unofficial].
      vii. If `f` is false (voxel is empty): Set `store = false` [S1, unofficial].
      viii. Flush runs if state changed or end of column: If `solid > 0` and `!store`: While `skip > 1023`: `slabs.push_back(1023)`; `skip -= 1023` [S1, unofficial]. While `solid > 63`: `slabs.push_back((63 << 10) | (skip & 1023))`; `solid -= 63`; `skip = 0`. `slabs.push_back((solid & 63) << 10 | (skip & 1023))`. `solid = 0`; `skip = 0` [S1, unofficial].
      ix. Update counters: If `store` is true: `solid++` [S1, unofficial]. Else: `skip++` [S1, unofficial].
   f. Finalize slab: `slabs[slab_len_pos] = slabs.size() - (slab_len_pos + 2)` [S1, unofficial]. `slabs[tex_len_pos] = texture.size()` [S1, unofficial]. Append `texture` entries to `slabs` [S1, unofficial].
3. Allocate `map4.map` of size $sx \cdot sz$ [S1, unofficial]. Copy `map` data [S1, unofficial].
4. Set `map4.slabs` to `slabs` data, `map4.slabs_size` to `slabs.size() * 2` [S1, unofficial].

**Formulas:**
- Voxel bit extraction: $f = \left( \text{mem}\left[\left(i + j \cdot sx + k \cdot sxy\right) \gg 3\right] \gg (i \ \& \ 7) \right) \ \& \ 1$, where $sxy = sx \cdot sy$ [S1, unofficial].
- Centroid normal: $\vec{c} = (-m_x, m_y, -m_z)$, normalized [S1, unofficial].
- Quantization: $n_x = \text{clamp}(\lfloor 127.0 \cdot c_x + 128.0 \rfloor, 0, 255)$ [S1, unofficial].
- Material extraction: $\text{mat} = (\text{bit}_1 \ | \ \text{bit}_2) \gg 8$, where $\text{bit}_1 = (\text{col}_1[\dots] \ \& \ 2^i) ? 256 : 0$, $\text{bit}_2 = (\text{col}_2[\dots] \ \& \ 2^i) ? 512 : 0$ [S1, unofficial].
- Normal perturbation: $\text{ny} = \left( \text{ny} \cdot \left( \left( \left( (i \cdot \text{scale}) \ \oplus \ (k \cdot \text{scale}) \right) \gg 4 \right) \ \& \ 15 + 15 \right) \right) \gg 5$ for $\text{mat}=0$ [S1, unofficial].
- RLE packing: $\text{entry} = (\text{solid} \ll 10) \ | \ (\text{skip} \ \& \ 1023)$ [S1, unofficial].

**Parameters:**
- `scale`: Integer multiplier, `1 << mip_lvl`. Used to modulate noise functions for normal perturbation [S1, unofficial].
- Max Solid Run: 63 (6 bits) [S1, unofficial].
- Max Skip Run: 1023 (10 bits) [S1, unofficial].
- Typical dimensions: $sx=1024, sy=256, sz=1024$ [S1, unofficial].

**Edge Cases and Pitfalls:**
- The Y-loop must iterate to $sy$ (inclusive) to ensure the final run of voxels is flushed [S1, unofficial].
- Boundary checks are required when sampling 26 neighbors during surface detection; out-of-bounds coordinates must be skipped to prevent segfaults [S1, unofficial].
- Solid runs exceeding 63 must be split into multiple RLE entries, resetting the skip counter to 0 each time [S1, unofficial].
- Skip runs exceeding 1023 must be split into multiple entries with `solid=0` [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import math

def rle4_compress(mem, col1, col2, sx, sy, sz, mip_lvl):
    scale = 1 << mip_lvl
    sxy = sx * sy
    map_offsets = [0] * (sx * sz)
    slabs = []

    for i in range(sx):
        for k in range(sz):
            idx = i + k * sx
            map_offsets[idx] = len(slabs)

            slab_len_pos = len(slabs)
            slabs.append(0)
            tex_len_pos = len(slabs)
            slabs.append(0)

            solid = 0
            skip = 0
            store = False
            texture = []

            for j in range(sy + 1):
                if j < sy:
                    voxel_idx = (i + j * sx + k * sxy) >> 3
                    bit_pos = i & 7
                    f = (mem[voxel_idx] >> bit_pos) & 1
                else:
                    f = 0

                if f:
                    mx, my, mz, cnt = 0, 0, 0, 0
                    for a in range(-1, 2):
                        for b in range(-1, 2):
                            for c in range(-1, 2):
                                nx, ny, nz = i + a, j + b, k + c
                                if 0 <= nx < sx and 0 <= ny < sy and 0 <= nz < sz:
                                    n_idx = (nx + ny * sx + nz * sxy) >> 3
                                    n_bit = nx & 7
                                    if (mem[n_idx] >> n_bit) & 1:
                                        mx += a
                                        my += b
                                        mz += c
                                        cnt += 1

                    nx_val, ny_val, nz_val = 128, 128, 128
                    if cnt < 27:
                        cx, cy, cz = -mx, my, -mz
                        len_sq = cx*cx + cy*cy + cz*cz
                        if len_sq > 0:
                            inv_len = 1.0 / (len_sq ** 0.5)
                            cx *= inv_len
                            cy *= inv_len
                            cz *= inv_len
                        nx_val = int(127.0 * cx + 128.0)
                        ny_val = int(127.0 * cy + 128.0)
                        nz_val = int(127.0 * cz + 128.0)

                    nx_val = max(0, min(255, nx_val))
                    ny_val = max(0, min(255, ny_val))
                    nz_val = max(0, min(255, nz_val))

                    bit1 = 256 if (col1[(i + j * sx + k * sxy) >> 3] >> (i & 7)) & 1 else 0
                    bit2 = 512 if (col2[(i + j * sx + k * sxy) >> 3] >> (i & 7)) & 1 else 0
                    mat = (bit1 | bit2) >> 8

                    rnd1 = math.sin(i * scale)
                    rnd2 = math.cos(k * scale)
                    rnd3 = math.sin(j * scale)
                    rnd4 = int(rnd1 + rnd2 + rnd3 + 3)

                    if mat == 0:
                        noise_factor = ((((i * scale) ^ (k * scale)) >> 4) & 15) + 15
                        ny_val = (ny_val * noise_factor) >> 5
                    elif mat == 3:
                        noise_factor = ((((i * scale) ^ (k * scale)) >> 3) & 15) + 25
                        ny_val = (ny_val * noise_factor) >> 5

                    ny_val = max(0, min(255, ny_val))
                    texture.append(ny_val | bit1 | bit2)
                    store = True
                else:
                    store = False

                if solid > 0 and not store:
                    while skip > 1023:
                        slabs.append(1023)
                        skip -= 1023
                    while solid > 63:
                        slabs.append((63 << 10) | (skip & 1023))
                        solid -= 63
                        skip = 0
                    slabs.append((solid & 63) << 10 | (skip & 1023))
                    solid = 0
                    skip = 0

                if store:
                    solid += 1
                else:
                    skip += 1

            slabs[slab_len_pos] = len(slabs) - (slab_len_pos + 2)
            slabs[tex_len_pos] = len(texture)
            slabs.extend(texture)

    map4 = {
        'sx': sx, 'sy': sy, 'sz': sz,
        'map': map_offsets,
        'slabs': slabs,
        'slabs_size': len(slabs) * 2
    }
    return map4
```

## Subtopics

### [Run-length encoded voxel runs](1209-run-length-encoded-voxel-runs.md)
General RLE for voxels. Choose when variable-length runs are acceptable and no fixed 16-bit constraint exists.

### [Voxel File I/O (binvox/rawvox)](1698-voxel-file-i-o-binvox-rawvox.md)
Standard file formats. Choose for storage/transfer, not real-time rendering.

### [Voxel Indexing and Bit Compression](1707-voxel-indexing-and-bit-compression.md)
General bit-packing. Choose for dense storage without run-length optimization.

### [Dense-grid to Voxlap-slab packing](2155-dense-grid-to-voxlap-slab-packing.md)
Packing for GPU textures. Choose when targeting texture-based rendering rather than splatting.

## Operating characteristics
- **Time Complexity**: Not given in the sources.
- **Memory Complexity**: Not given in the sources.
- **Performance**: Not given in the sources.
- **Strengths**: Preserves per-voxel color/intensity. Fast decoding for splatting. Compact for sparse data [S1, unofficial].
- **Limitations**: Fixed run lengths (6/10 bits) may waste space if runs are very long or very short. Normal perturbation is heuristic [S1, unofficial].
- **When to use**: Real-time voxel splatting with sparse data [S1, unofficial].
- **When not to use**: Dense volumes, or when exact geometric normals are required without perturbation [S1, unofficial].

## Background
RLE4 Compression was introduced in the OpenGL Voxel Splatting project by sp4cerat [S1, unofficial]. It is designed for real-time rendering of voxel data using OpenGL, specifically targeting the `GLX` and `GL` extensions for texture mapping [S1, unofficial]. The method is part of a pipeline that includes voxelization and binary RLE encoding, optimizing the data for GPU splatting [S1, unofficial].

## Sources
[S1] sp4cerat/OpenGL-Voxel-Splatting, https://github.com/sp4cerat/OpenGL-Voxel-Splatting, fetched 2026-10-27 (unofficial)
