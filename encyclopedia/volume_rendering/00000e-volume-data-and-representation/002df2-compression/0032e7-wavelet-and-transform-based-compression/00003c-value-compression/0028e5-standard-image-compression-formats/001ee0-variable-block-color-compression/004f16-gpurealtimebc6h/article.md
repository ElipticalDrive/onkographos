# GPURealTimeBC6H
Also called: Real-time BC6H compression on GPU
## Overview
GPURealTimeBC6H is a GPU-based real-time compressor for the BC6H texture format [S1, unofficial]. It solves the problem of high-quality HDR texture compression by pruning the BC6H block mode search space to specific modes tailored for speed or quality presets [S1, unofficial]. It sets itself apart from offline compressors by performing all arithmetic in log2 space to optimize for perceptual error after tone mapping, and by restricting the search to modes 11 (fast) or 2 and 6 (quality) [S1, unofficial].
## How it works
**Inputs and Outputs:**
Input: A 4x4 block of HDR texels (typically 32-bit floating point per channel).
Output: A BC6H compressed block (128 bits).

**Data Structures:**
The BC6H format uses 128 bits per 4x4 block. The specific bit layout depends on the selected mode.
- Mode 11: Uses a single partition (all 16 texels share the same endpoints).
- Modes 2 and 6: Use two partitions (8 texels each).

**Parameters:**
- `QUALITY`: A macro flag. If defined, the compressor uses Quality mode; otherwise, it uses Fast mode [S1, unofficial].
- `LOG2_SPACE`: All arithmetic is performed in log2 space [S1, unofficial].

**Steps:**

1. **Log2 Conversion:** Convert all input texel color values $C$ to log2 space: $L = \log_2(C)$. Handle zero or negative values appropriately (e.g., clamp to a minimum representable value) [G].

2. **Mode Selection:**
   - If `QUALITY` is not defined (Fast Mode): Proceed to Step 3.
   - If `QUALITY` is defined (Quality Mode): Proceed to Step 4.

3. **Fast Mode (Mode 11):**
   a. Compute the color bounding box of the 16 texels in log2 space.
   b. Order the colors by a diagonal of this bounding box.
   c. Use least squares fit to find the optimal endpoints $E_0$ and $E_1$ for the line in log2 space [S1, unofficial].
   d. Encode the block using BC6H Mode 11 with endpoints $E_0$ and $E_1$.

4. **Quality Mode (Modes 2 and 6):**
   a. **Pass 1 (Partition Search):**
      i. Iterate through all valid partitions for Modes 2 and 6.
      ii. For each partition, compute the color bounding box for each of the two subsets of texels.
      iii. Calculate the error for each partition as the sum of distances of every texel from its partition's line in log2 space.
      iv. Select the partition with the minimum error.
   b. **Pass 2 (Encoding):**
      i. Encode the best partition using an approach similar to Fast Mode (least squares fit for endpoints) [S1, unofficial].
      ii. Encode the block using the corresponding BC6H Mode (2 or 6).

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def compress_block(texels_4x4, quality_mode=False):
    # Convert to log2 space
    # Handle non-positive values by clamping to a small epsilon or minimum representable log2 value
    log_texels = [log2_safe(c) for c in texels_4x4]
    
    if not quality_mode:
        # Fast Mode: Mode 11
        # 1. Compute bounding box of log_texels
        min_val = min(log_texels)
        max_val = max(log_texels)
        
        # 2. Order colors by diagonal (simplified: sort by value)
        # Note: The source mentions ordering by diagonal, but for Mode 11 (single line),
        # sorting by the scalar value along the diagonal is sufficient for least squares.
        sorted_texels = sorted(log_texels)
        
        # 3. Least squares fit for endpoints E0, E1
        # For a single line, E0 and E1 are typically the min and max of the fitted line
        # or derived from the bounding box. The source says "using least square fit to find optimal endpoints".
        # A simple approximation for Mode 11 is to use the min and max of the sorted texels
        # or the bounds of the bounding box.
        e0 = min_val
        e1 = max_val
        
        # 4. Encode BC6H Mode 11
        return encode_bc6h_mode11(e0, e1)
    else:
        # Quality Mode: Modes 2 and 6
        best_partition = None
        best_error = infinity
        
        # Iterate through partitions for Mode 2 and Mode 6
        for partition in get_partitions_for_modes([2, 6]):
            # Split texels into two groups based on partition
            group1 = [log_texels[i] for i in partition.group1_indices]
            group2 = [log_texels[i] for i in partition.group2_indices]
            
            # Compute bounding box for each group
            min1 = min(group1)
            max1 = max(group1)
            min2 = min(group2)
            max2 = max(group2)
            
            # Calculate error as distance of every texel from that line
            # For each group, the line is defined by min and max.
            # Error is sum of squared distances from the line.
            # For a 1D line in log space, distance of a point x from line [a, b] is 0 if x is on the line.
            # However, the source says "distance of every texel from that line".
            # In 3D color space, this is the perpendicular distance.
            # Since we are in log2 space and treating it as a line, we approximate.
            # A common approximation is the sum of squared differences from the fitted line.
            # For simplicity, we assume the error is the sum of squared distances from the bounding box diagonal.
            error1 = sum((x - lerp(min1, max1, t))^2 for x in group1) # Placeholder for actual 3D distance
            error2 = sum((x - lerp(min2, max2, t))^2 for x in group2)
            total_error = error1 + error2
            
            if total_error < best_error:
                best_error = total_error
                best_partition = partition
        
        # Encode best partition
        # Similar to Fast Mode, find endpoints for each group
        g1_indices = best_partition.group1_indices
        g2_indices = best_partition.group2_indices
        g1 = [log_texels[i] for i in g1_indices]
        g2 = [log_texels[i] for i in g2_indices]
        
        e0 = min(g1)
        e1 = max(g1)
        e2 = min(g2)
        e3 = max(g2)
        
        # Determine if Mode 2 or 6 was used
        mode = best_partition.mode
        return encode_bc6h_mode(mode, e0, e1, e2, e3)

def log2_safe(x):
    if x <= 0:
        return -128.0 # Minimum representable value [G]
    return math.log2(x)
```

**Edge Cases and Pitfalls:**
- Zero or negative texel values: Log2 is undefined. Clamp to a minimum representable value [G].
- Precision: Using log2 space may introduce precision issues for very small or very large values. The source does not specify the floating-point precision used for intermediate calculations [S1, unofficial].
- Boundary conditions: The bounding box computation must handle cases where all texels are identical (min == max) [G].

## Operating characteristics
**Time and Memory Complexity:**
- Fast Mode: Extremely fast. Compresses a 256x256x6 cubemap in 0.02ms on an NV P4000 [S1, unofficial].
- Quality Mode: Slower but still real-time. Compresses a 256x256x6 cubemap in 0.528ms on an NV P4000 [S1, unofficial].
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
- Use Fast Mode when speed is critical and quality comparable to offline "fast" presets is acceptable.
- Use Quality Mode when higher fidelity is needed and slightly more time is available.

## Background
GPURealTimeBC6H is an unofficial implementation of a real-time BC6H compressor [S1, unofficial]. It builds on techniques from "Real-Time DXT Compression" by J.M.P. van Waveren (2006) and "High Quality DXT Compression using CUDA" by Ignacio Castaño (2007) [S1, unofficial]. It is used in several released AA/AAA games [S1, unofficial].

## Sources
[S1] Real-time BC6H compression on GPU, https://github.com/knarkowicz/GPURealTimeBC6H, fetched 2026-10-27 (unofficial)
