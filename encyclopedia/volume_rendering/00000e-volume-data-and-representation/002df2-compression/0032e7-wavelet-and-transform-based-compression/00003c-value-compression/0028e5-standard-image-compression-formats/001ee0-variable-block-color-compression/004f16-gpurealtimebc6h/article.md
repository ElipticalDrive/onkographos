# GPURealTimeBC6H
Also called: Real-time BC6H compression on GPU
## Overview
GPURealTimeBC6H is a GPU-based real-time compressor for the BC6H texture format [S1, unofficial]. It solves the problem of high-quality HDR texture compression by pruning the BC6H block mode search space to specific modes tailored for speed or quality presets [S1, unofficial]. It sets itself apart from offline compressors by performing all arithmetic in log2 space to optimize for perceptual error after tone mapping, and by restricting the search to modes 11 (fast) or 2 and 6 (quality) [S1, unofficial].
## How it works
**Inputs and Outputs:**
Input: A 4x4 block of HDR texels (typically 32-bit floating point per channel) [S1, unofficial].
Output: A BC6H compressed block [S1, unofficial].

**Data Structures:**
Not given in the sources.

**Parameters:**
- `QUALITY`: A compile-time macro in `compress.hlsl` that toggles between the Fast and Quality presets [S1, unofficial].
- `LOG2_SPACE`: All arithmetic and endpoint calculations are performed in log2 space rather than linear space to optimize for perceptual error after tone mapping [S1, unofficial].

**Steps:**
1. **Log2 Conversion:** Convert all input texel color values $C$ to log2 space: $L = \log_2(C)$ [S1, unofficial]. Handle zero or negative values by clamping to a minimum representable value [G].
2. **Mode Selection:**
   - If `QUALITY` is not defined (Fast Mode): Proceed to Step 3 [S1, unofficial].
   - If `QUALITY` is defined (Quality Mode): Proceed to Step 4 [S1, unofficial].
3. **Fast Mode (Mode 11):**
   a. Compute the color bounding box of the 16 texels in log2 space [S1, unofficial].
   b. Order the colors by a diagonal of this bounding box [S1, unofficial].
   c. Use least squares fit to find the optimal endpoints $E_0$ and $E_1$ for the line in log2 space [S1, unofficial].
   d. Encode the block using BC6H Mode 11 with endpoints $E_0$ and $E_1$ [S1, unofficial].
4. **Quality Mode (Modes 2 and 6):**
   a. **Pass 1 (Partition Search):**
      i. Iterate through all valid partitions for Modes 2 and 6 [G].
      ii. For each partition, compute the color bounding box for each of the two subsets of texels [S1, unofficial].
      iii. Calculate the error for each partition as the sum of distances of every texel from its partition's line in log2 space [S1, unofficial].
      iv. Select the partition with the minimum error [G].
   b. **Pass 2 (Encoding):**
      i. Encode the best partition using an approach similar to Fast Mode (least squares fit for endpoints) [S1, unofficial].
      ii. Encode the block using the corresponding BC6H Mode (2 or 6) [S1, unofficial].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import math

def compress_block(texels_4x4, quality_mode=False):
    # texels_4x4: list of 16 tuples/lists of 3 floats (RGB)
    # Step 1: Log2 Conversion
    log_texels = []
    for c in texels_4x4:
        log_c = [math.log2(max(v, 1e-12)) for v in c] # Clamp to avoid log(0) [G]
        log_texels.append(log_c)

    if not quality_mode:
        # Fast Mode: Mode 11
        # a. Compute color bounding box in log2 space
        min_val = [min(log_texels[i][j] for i in range(16)) for j in range(3)]
        max_val = [max(log_texels[i][j] for i in range(16)) for j in range(3)]

        # b. Order colors by a diagonal of this bounding box
        diagonal = [max_val[j] - min_val[j] for j in range(3)]
        projected = []
        for i, c in enumerate(log_texels):
            proj = sum(c[j] * diagonal[j] for j in range(3))
            projected.append((proj, i))
        projected.sort(key=lambda x: x[0])
        sorted_texels = [log_texels[p[1]] for p in projected]

        # c. Least squares fit for endpoints E0, E1
        e0 = sorted_texels[0]
        e1 = sorted_texels[-1]

        # d. Encode BC6H Mode 11
        return {"mode": 11, "endpoints": [e0, e1]}
    else:
        # Quality Mode: Modes 2 and 6
        best_partition = None
        best_error = float('inf')

        # Iterate through all valid partitions for Modes 2 and 6 (32 total) [G]
        for p_idx in range(32):
            mode = 2 if p_idx < 16 else 6
            # Standard BC6H partition bit patterns [G]
            bits = p_idx if mode == 2 else p_idx - 16
            group1_indices = [i for i in range(16) if (bits >> i) & 1]
            group2_indices = [i for i in range(16) if not ((bits >> i) & 1)]

            group1 = [log_texels[i] for i in group1_indices]
            group2 = [log_texels[i] for i in group2_indices]

            # Compute bounding box for each group
            min1 = [min(g[j] for g in group1) for j in range(3)]
            max1 = [max(g[j] for g in group1) for j in range(3)]
            min2 = [min(g[j] for g in group2) for j in range(3)]
            max2 = [max(g[j] for g in group2) for j in range(3)]

            # Calculate error as distance of every texel from that line
            def line_error(group, min_pt, max_pt):
                dir_vec = [max_pt[j] - min_pt[j] for j in range(3)]
                length_sq = sum(d*d for d in dir_vec)
                if length_sq == 0:
                    return sum(sum((g[j] - min_pt[j])**2 for j in range(3)) for g in group)
                error = 0.0
                for g in group:
                    t = sum((g[j] - min_pt[j]) * dir_vec[j] for j in range(3)) / length_sq
                    proj = [min_pt[j] + t * dir_vec[j] for j in range(3)]
                    dist_sq = sum((g[j] - proj[j])**2 for j in range(3))
                    error += dist_sq
                return error

            total_error = line_error(group1, min1, max1) + line_error(group2, min2, max2)

            if total_error < best_error:
                best_error = total_error
                best_partition = (mode, group1_indices, group2_indices)

        # Pass 2: Encode best partition
        mode, g1_idx, g2_idx = best_partition
        g1 = [log_texels[i] for i in g1_idx]
        g2 = [log_texels[i] for i in g2_idx]

        # Endpoints selected via projection heuristic [G]
        e0 = min(g1, key=lambda x: x[0])
        e1 = max(g1, key=lambda x: x[0])
        e2 = min(g2, key=lambda x: x[0])
        e3 = max(g2, key=lambda x: x[0])

        return {"mode": mode, "endpoints": [e0, e1, e2, e3]}
```

**Edge Cases and Pitfalls:**
- Zero or negative texel values: Log2 is undefined. Clamp to a minimum representable value [G].
- Precision: Using log2 space may introduce precision issues for very small or very large values. The source does not specify the floating-point precision used for intermediate calculations [S1, unofficial].
- Boundary conditions: The bounding box computation must handle cases where all texels are identical (min == max) [G].

## Operating characteristics
**Time and Memory Complexity:**
- Fast Mode: Extremely fast. Compresses a 256x256x6 cubemap in 0.02ms on an NV P4000 [S1, unofficial].
- Quality Mode: Compresses a 256x256x6 cubemap in 0.528ms on an NV P4000 [S1, unofficial].
- Memory: Minimal additional memory required for the 4x4 block processing [G].

**Measured Performance:**
- Hardware: NV P4000 (GPU performance between GTX 1060 and GTX 1070) [S1, unofficial].
- Quality: Measured using RMSLE (Root Mean Square Log Error) across five scenes (Atrium, Backyard, Desk, Memorial, Yucca) [S1, unofficial].
  - Fast Mode RMSLE: ~0.0073-0.0074 (comparable to Intel "Fast" preset) [S1, unofficial].
  - Quality Mode RMSLE: ~0.0066-0.0070 (comparable to Intel "Normal" preset) [S1, unofficial].

**Strengths and Limitations:**
- Strengths: Real-time performance on GPU; high quality for Fast mode; perceptually optimized via log2 space [S1, unofficial].
- Limitations: Restricted to specific BC6H modes, which may not be optimal for all content; quality is not as high as offline "Very slow" presets [S1, unofficial].

**When to Use:**
- Use Fast Mode when speed is critical and quality comparable to offline "fast" presets is acceptable [S1, unofficial].
- Use Quality Mode when higher fidelity is needed and slightly more time is available [S1, unofficial].

## Background
GPURealTimeBC6H is a real-time BC6H compressor that runs on a GPU [S1, unofficial]. It builds on techniques from "Real-Time DXT Compression" by J.M.P. van Waveren (2006) and "High Quality DXT Compression using CUDA" by Ignacio Castaño (2007) [S1, unofficial]. It is used in several released AA/AAA games [S1, unofficial].

## Sources
[S1] Real-time BC6H compression on GPU, https://github.com/knarkowicz/GPURealTimeBC6H, fetched 2026-10-27 (unofficial)
