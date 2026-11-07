# Pre-processing for Half Resolution
Also called: Half-resolution volume construction
## Overview
This method constructs a half-resolution volumetric representation from a full-resolution input to reduce data size while preserving visual quality and avoiding the information loss associated with direct subsampling [S1]. It differs from its sibling [VoxelGrid Down Sample](1014-voxelgrid-down-sample.md), which uses max pooling, by explicitly filling "holes" (transparent voxels in the downsampled grid) with median values from adjacent full-resolution neighbors to prevent occlusion artifacts [S1]. Unlike [Frequency-Domain Downsampling](6578-frequency-domain-downsampling.md), which relies on low-pass filtering in the frequency domain, this approach operates directly in the spatial domain using a specific subsampling pattern and local median replacement [S1].
## How it works
**Inputs:**
- Full-resolution volume $V_{full}$ of dimensions $H \times W \times Z$.
- A definition of "transparent" voxels (e.g., background or invisible voxels) [S1].

**Outputs:**
- Pre-processed half-resolution volume $V_{half}$ of dimensions $\lceil H/2 \rceil \times \lceil W/2 \rceil \times \lceil Z/2 \rceil$.

**Data Structures:**
- $V_{full}$: 3D array of voxel values (color/density).
- $V_{half}$: 3D array of voxel values, initialized to transparent or empty.

**Steps:**
1. **Subsampling:** Create $V_{half}$ by selecting voxels from $V_{full}$ at specific indices. For each integer coordinate $(\hat{i}, \hat{j}, \hat{k})$ where $\hat{i}, \hat{j}, \hat{k} \in \{1, 2, 3, ...\}$, map the voxel at $V_{full}[2\hat{i}-1, 2\hat{j}-1, 2\hat{k}-1]$ to $V_{half}[\hat{i}, \hat{j}, \hat{k}]$ [S1]. The indices must satisfy $1 \le 2\hat{i}-1 \le H$, $1 \le 2\hat{j}-1 \le W$, and $1 \le 2\hat{k}-1 \le Z$ [S1].

2. **Hole Filling:** Iterate through every position $(\hat{i}, \hat{j}, \hat{k})$ in $V_{half}$. If the voxel $V_{half}[\hat{i}, \hat{j}, \hat{k}]$ is transparent, examine the eight adjacent voxels in the full-resolution volume $V_{full}$ [S1]. These adjacent voxels correspond to the neighbors of the subsampled position in the original grid. Specifically, if the subsampled voxel is at $(x, y, z) = (2\hat{i}-1, 2\hat{j}-1, 2\hat{k}-1)$, the eight adjacent voxels are those at coordinates $(x+\delta_x, y+\delta_y, z+\delta_z)$ where $\delta_x, \delta_y, \delta_z \in \{-1, 1\}$ [S1].

3. **Median Replacement:** If any of the eight adjacent full-resolution voxels is not transparent, replace the transparent voxel in $V_{half}[\hat{i}, \hat{j}, \hat{k}]$ with the median value of the non-transparent voxels among those eight adjacent positions [S1]. If all eight are transparent, the voxel remains transparent.

4. **Boundary Removal:** Remove any newly inserted voxels (those filled in Step 3) that are located outside the original volume bounds or would occlude/affect points of the original full-resolution volume [S1]. This ensures the pre-processed data does not introduce artifacts when viewed at full resolution [S1].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def preprocess_half_resolution(V_full, H, W, Z, is_transparent):
    # Initialize half-resolution volume
    H_half = (H + 1) // 2
    W_half = (W + 1) // 2
    Z_half = (Z + 1) // 2
    V_half = [[[[None for _ in range(Z_half)] for _ in range(W_half)] for _ in range(H_half)]]

    # Step 1: Subsampling
    for i_hat in range(1, H_half + 1):
        for j_hat in range(1, W_half + 1):
            for k_hat in range(1, Z_half + 1):
                x = 2 * i_hat - 1
                y = 2 * j_hat - 1
                z = 2 * k_hat - 1
                # Check bounds for full-resolution indices
                if 1 <= x <= H and 1 <= y <= W and 1 <= z <= Z:
                    V_half[i_hat-1][j_hat-1][k_hat-1] = V_full[x-1][y-1][z-1]
                else:
                    V_half[i_hat-1][j_hat-1][k_hat-1] = None # Out of bounds

    # Step 2 & 3: Hole Filling
    for i_hat in range(1, H_half + 1):
        for j_hat in range(1, W_half + 1):
            for k_hat in range(1, Z_half + 1):
                x = 2 * i_hat - 1
                y = 2 * j_hat - 1
                z = 2 * k_hat - 1
                
                # If current half-res voxel is transparent/None, check neighbors
                if V_half[i_hat-1][j_hat-1][k_hat-1] is None or is_transparent(V_half[i_hat-1][j_hat-1][k_hat-1]):
                    neighbors = []
                    # Check 8 adjacent voxels in full-res grid
                    for dx in [-1, 1]:
                        for dy in [-1, 1]:
                            for dz in [-1, 1]:
                                nx, ny, nz = x + dx, y + dy, z + dz
                                if 1 <= nx <= H and 1 <= ny <= W and 1 <= nz <= Z:
                                    neighbor_val = V_full[nx-1][ny-1][nz-1]
                                    if not is_transparent(neighbor_val):
                                        neighbors.append(neighbor_val)
                    
                    if neighbors:
                        # Replace with median of non-transparent neighbors
                        median_val = median(neighbors)
                        V_half[i_hat-1][j_hat-1][k_hat-1] = median_val
                    # Else remains transparent/None

    # Step 4: Boundary Removal (Implementation note: The source states removal of voxels 
    # located outside the volume. In a fixed-size array, this typically means 
    # ensuring no out-of-bounds access or logical removal from the active set. 
    # Since V_half is derived from valid subsampling indices, "outside" likely refers 
    # to logical validity or specific occlusion checks not fully detailed in text [S1].
    # We assume the array structure is maintained but invalid entries are handled.)
    
    return V_half

def median(values):
    sorted_vals = sorted(values)
    n = len(sorted_vals)
    if n % 2 == 1:
        return sorted_vals[n // 2]
    else:
        return (sorted_vals[n // 2 - 1] + sorted_vals[n // 2]) / 2
```

**Parameters:**
- Subsampling factor: 2 (half resolution) [S1].
- Neighbor count: 8 (adjacent voxels in 3D space) [S1].

**Edge Cases and Pitfalls:**
- **Boundary Conditions:** The subsampling formula $2\hat{i}-1$ ensures odd indices are selected. If the volume dimensions are even, the last index in each dimension is skipped in the initial subsampling step [S1].
- **Transparency Definition:** The method relies on a clear definition of "transparent" voxels (e.g., background) [S1].
- **Occlusion:** The removal step ensures new voxels do not occlude original points, but the exact mechanism for "removing" voxels from a fixed array is not specified beyond logical exclusion [S1].

## Subtopics

### [Frequency-Domain Downsampling](6578-frequency-domain-downsampling.md)
A non-learnable pre-processing method that reduces resolution by transforming data to the frequency domain, applying a low-pass filter, and transforming back. It avoids aliasing artifacts and allows controllable downsampling scales without training, unlike the spatial median approach of this method.

## Operating characteristics
- **Performance:** The pre-processed half-resolution volume achieves a PSNR of 32.08 dB for full-resolution reconstruction [S1]. This is significantly higher than direct subsampling (22.75 dB), linear interpolation (27.44 dB), and nearest interpolation (26.03 dB) [S1].
- **Complexity:** The method involves a single pass for subsampling and a second pass for hole filling, where each transparent voxel checks up to 8 neighbors. The complexity is linear with respect to the number of voxels in the half-resolution volume, multiplied by a constant factor for neighbor checks [S1].
- **Strengths:** Eliminates holes in synthesized views and improves image quality by preserving edge sharpness and homogeneity [S1].
- **Limitations:** Requires a definition of transparent voxels and may introduce slight blurring due to median filtering [S1].
## Background
This method was introduced in the context of color volumetric compression for realistic view synthesis applications [S1]. It addresses the problem of information loss when reducing volumetric data resolution for efficient transmission or storage [S1].
## Sources
[S1] Colour volumetric compression for realistic view synthesis applications, https://doi.org/10.1007/s11042-010-0484-4, fetched 2026-10-26
