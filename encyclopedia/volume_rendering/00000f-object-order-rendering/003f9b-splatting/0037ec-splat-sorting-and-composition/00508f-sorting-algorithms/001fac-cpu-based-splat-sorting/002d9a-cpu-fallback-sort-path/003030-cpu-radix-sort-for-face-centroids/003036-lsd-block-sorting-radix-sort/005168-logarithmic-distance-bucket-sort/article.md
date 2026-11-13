# Logarithmic Distance Bucket Sort

Also called: Back-to-Front Voxel Sorting, DistanceSorter

## Overview
Logarithmic Distance Bucket Sort is a CPU-based sorting algorithm for volume rendering voxels that orders them back-to-front relative to the camera position. It solves the depth-sorting requirement for alpha blending or depth peeling by mapping voxel distances into fixed logarithmic buckets. It differs from Morton-order-based splat sorting (e.g., [SVRaster](52-svraster.md)) by relying solely on radial distance rather than ray-direction-dependent spatial locality, making it robust for arbitrary camera orientations but potentially less cache-friendly for static scenes.

## How it works
**Inputs:**
- `positions`: `Float32Array` of voxel centers, interleaved as `[x1, y1, z1, x2, y2, z2, ...]`.
- `cameraPosition`: `[number, number, number]` representing the camera origin.
- `outIndices`: Optional pre-allocated `Uint32Array` for output.

**Outputs:**
- `Uint32Array` of indices into `positions`, sorted such that voxels with larger distances from the camera appear first (back-to-front).

**Data Structures:**
The implementation maintains static state to avoid reallocation between frames:
- `counts`: `Uint32Array` of size `BUCKET_COUNT` (65536). Stores bucket occupancy during the first pass.
- `starts`: `Uint32Array` of size `BUCKET_COUNT`. Stores the starting index for each bucket in the output array.
- `lastCameraPosition`: `[number, number, number]` or `null`. Stores the previous camera position for early-exit optimization.

**Parameters:**
- `BUCKET_COUNT`: 65536 ($256 \times 256$).

**Algorithm Steps:**

1.  **Early Exit Check:**
    If `lastCameraPosition` exists, `outIndices` is provided, and `outIndices.length` equals `numVoxels`:
    - Calculate the dot product of `lastCameraPosition` and `cameraPosition`.
    - Calculate the squared lengths of both vectors.
    - Compute the cosine of the angle: $\cos \theta = \frac{\mathbf{c}_{last} \cdot \mathbf{c}_{curr}}{\|\mathbf{c}_{last}\| \|\mathbf{c}_{curr}\|}$.
    - If $| \cos \theta - 1 | < 0.01$, return `outIndices` immediately. This assumes the sort order remains valid if the camera direction has not changed significantly. [S1, unofficial]

2.  **Initialization:**
    - Determine `numVoxels` as `positions.length / 3`.
    - Allocate or reuse `indices` array of size `numVoxels`.
    - If `counts` or `starts` are null or have incorrect length, allocate them as `Uint32Array` of size `BUCKET_COUNT`.
    - Otherwise, clear `counts` by filling with zeros. [S1, unofficial]

3.  **Log Distance Calculation:**
    - Initialize `minDist` to $\infty$ and `maxDist` to $-\infty$.
    - For each voxel $i$ from $0$ to `numVoxels - 1`:
        - Extract coordinates $(x, y, z)$.
        - Compute squared Euclidean distance: $d^2 = (x - c_x)^2 + (y - c_y)^2 + (z - c_z)^2$.
        - Compute logarithmic distance: $L_i = \log(d^2)$.
        - Store $L_i$ in a temporary `logDistances` array.
        - Update `minDist` and `maxDist` based on $L_i$. [S1, unofficial]

4.  **Bucket Scaling:**
    - Calculate range: $R = \max(1, \text{maxDist} - \text{minDist})$. The minimum range is 1 to prevent division by zero. [S1, unofficial]
    - Calculate inverse scale factor: $S = \frac{\text{BUCKET\_COUNT} - 1}{R}$. [S1, unofficial]

5.  **First Pass (Counting):**
    - For each voxel $i$:
        - Calculate bucket index: $b_i = \min(\text{BUCKET\_COUNT} - 1, \lfloor (L_i - \text{minDist}) \times S \rfloor)$.
        - Increment `counts[b_i]`. [S1, unofficial]

6.  **Calculate Bucket Starts:**
    - Initialize `position = 0`.
    - Iterate $j$ from `BUCKET_COUNT - 1` down to $0$:
        - Set `starts[j] = position`.
        - Update `position += counts[j]`.
    - This ensures back-to-front ordering (higher distance buckets are placed earlier in the array). [S1, unofficial]

7.  **Reset Counts:**
    - Clear `counts` array by filling with zeros. [S1, unofficial]

8.  **Second Pass (Distribution):**
    - For each voxel $i$:
        - Calculate bucket index $b_i$ using the same formula as Step 5.
        - Calculate output position: $p = \text{starts}[b_i] + \text{counts}[b_i]$.
        - Set `indices[p] = i`.
        - Increment `counts[b_i]`. [S1, unofficial]

9.  **State Update:**
    - Store `cameraPosition` in `lastCameraPosition` for the next frame. [S1, unofficial]

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import math

BUCKET_COUNT = 65536
counts = None
starts = None
last_camera_position = None

def sort_voxels(positions, camera_position, out_indices=None):
    global counts, starts, last_camera_position
    
    num_voxels = len(positions) // 3
    
    # Early exit optimization
    if (last_camera_position is not None and 
        out_indices is not None and 
        len(out_indices) == num_voxels):
        
        lx, ly, lz = last_camera_position
        cx, cy, cz = camera_position
        
        dot = lx * cx + ly * cy + lz * cz
        len_sq1 = lx * lx + ly * ly + lz * lz
        len_sq2 = cx * cx + cy * cy + cz * cz
        
        # Avoid sqrt if possible, but source uses sqrt for normalization
        # cos_theta = dot / sqrt(len_sq1 * len_sq2)
        # Check if cos_theta is close to 1
        # |cos_theta - 1| < 0.01
        # Equivalent to: 1 - cos_theta < 0.01
        # 1 - dot/sqrt(...) < 0.01
        
        # Note: Source calculates abs(dot / sqrt(...) - 1) < 0.01
        # We must handle potential division by zero if camera is at origin, 
        # though typically camera is not at origin.
        # [G] Guard against zero-length camera vectors.
        if len_sq1 == 0 or len_sq2 == 0:
            pass # Proceed to sort
        else:
            cos_theta = dot / math.sqrt(len_sq1 * len_sq2)
            if abs(cos_theta - 1.0) < 0.01:
                return out_indices

    # Initialize output
    if out_indices is not None and len(out_indices) == num_voxels:
        indices = out_indices
    else:
        indices = [0] * num_voxels # Uint32Array equivalent

    # Initialize static arrays
    if counts is None or starts is None or len(counts) != BUCKET_COUNT:
        counts = [0] * BUCKET_COUNT
        starts = [0] * BUCKET_COUNT
    else:
        # Clear counts
        for i in range(BUCKET_COUNT):
            counts[i] = 0

    # Calculate log distances
    log_distances = [0.0] * num_voxels
    min_dist = float('inf')
    max_dist = float('-inf')
    
    cx, cy, cz = camera_position
    
    for i in range(num_voxels):
        x = positions[i * 3]
        y = positions[i * 3 + 1]
        z = positions[i * 3 + 2]
        
        dx = x - cx
        dy = y - cy
        dz = z - cz
        
        dist_sq = dx * dx + dy * dy + dz * dz
        log_dist = math.log(dist_sq)
        
        log_distances[i] = log_dist
        
        if log_dist < min_dist:
            min_dist = log_dist
        if log_dist > max_dist:
            max_dist = log_dist

    # Calculate scaling
    log_range = max_dist - min_dist
    if log_range == 0:
        log_range = 1 # Avoid division by zero
        
    dist_inv = (BUCKET_COUNT - 1) / log_range

    # First pass: Count
    for i in range(num_voxels):
        # floor((log_dist - min_dist) * dist_inv)
        bucket_idx = int((log_distances[i] - min_dist) * dist_inv)
        if bucket_idx >= BUCKET_COUNT:
            bucket_idx = BUCKET_COUNT - 1
        counts[bucket_idx] += 1

    # Calculate bucket starts (back-to-front)
    position = 0
    for i in range(BUCKET_COUNT - 1, -1, -1):
        starts[i] = position
        position += counts[i]

    # Reset counts for second pass
    for i in range(BUCKET_COUNT):
        counts[i] = 0

    # Second pass: Distribute
    for i in range(num_voxels):
        bucket_idx = int((log_distances[i] - min_dist) * dist_inv)
        if bucket_idx >= BUCKET_COUNT:
            bucket_idx = BUCKET_COUNT - 1
            
        pos = starts[bucket_idx] + counts[bucket_idx]
        indices[pos] = i
        counts[bucket_idx] += 1

    # Update last camera position
    last_camera_position = [cx, cy, cz]
    
    return indices
```

## Operating characteristics
- **Time Complexity:** $O(N + B)$, where $N$ is the number of voxels and $B$ is `BUCKET_COUNT` (65536). The algorithm performs two linear passes over the voxels and one linear pass over the buckets.
- **Memory Complexity:** $O(N + B)$. Requires storage for `logDistances` ($N$ floats), `indices` ($N$ integers), and static `counts`/`starts` arrays ($B$ integers each).
- **Performance:** The use of logarithmic distance mapping compresses the dynamic range of distances, allowing a fixed number of buckets to provide reasonable depth resolution across varying scene scales. The early-exit optimization significantly reduces cost when the camera is static or moving very slightly. [S1, unofficial]
- **Strengths:** Simple implementation, low memory overhead compared to full radix sorts, effective early-exit for static scenes.
- **Limitations:** Does not account for spatial locality (unlike Morton/Z-order sorting), which may result in poorer cache performance during the subsequent splatting/composition phase compared to spatially coherent sorts. Sorting is strictly based on distance, so voxels at the same distance are not ordered relative to each other. [S1, unofficial]

## Background
This method is implemented in the unofficial WebGL port of Nvidia's SVRaster paper, specifically in the `DistanceSorter` class. It is designed as a CPU fallback or primary sorter for 3D Gaussian Splatting rendering in WebGL contexts where GPU-based sorting might be complex or unavailable. It contrasts with the SVRaster paper's primary suggestion of using ray direction-dependent Morton ordering. [S1, unofficial]

## Sources
[S1] Gaussian Splatting Alternative: WebGL Implementation of Nvidia's SVRaster, https://github.com/samuelm2/svraster-webgl, fetched 2026-10-26 (unofficial)
