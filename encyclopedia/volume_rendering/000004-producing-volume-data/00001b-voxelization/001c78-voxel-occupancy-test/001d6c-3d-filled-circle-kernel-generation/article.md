# 3D Filled Circle Kernel Generation

Also called: 3D Binary Sphere Kernel, 3D Radial Mask Generation

## Overview
This method generates a 3D binary kernel (mask) where voxels within a specified Euclidean distance from the center are set to 1, and others are set to 0. It solves the problem of defining a discrete spherical occupancy region for voxel-based operations, such as smoothing, collision detection, or volume initialization. It differs from its sibling [Point-in-Box Inclusion Test](1400-point-in-box-inclusion-test.md) by using radial distance rather than axis-aligned bounds, and from its sibling [Voxel Filled Check](1705-voxel-filled-check.md) by generating a full kernel array rather than testing a single point.

## How it works
**Inputs:**
- `size`: Integer dimension of the cubic kernel (e.g., 3 for a 3x3x3 grid).
- `radius`: Float radius defining the sphere boundary.

**Outputs:**
- `kernel`: A 3D array of shape `(size, size, size)` with dtype boolean or integer (0/1).

**Data Structures:**
- The kernel is stored as a dense 3D array. Coordinates are indexed from `0` to `size-1`.

**Steps:**
1. Define the coordinate range: Create a 1D linear space of integers from `-(size // 2)` to `size // 2`. Let this array be $L$.
2. Generate meshgrid: Create a 3D meshgrid from $L$ for axes $x, y, z$. This produces three 3D arrays $X, Y, Z$ of shape `(size, size, size)`, where each element represents the coordinate offset from the center.
3. Compute radial distance: Calculate the Euclidean distance $d$ from the center for each voxel:
   $$ d = \sqrt{X^2 + Y^2 + Z^2} $$
4. Apply threshold: Set the kernel value to 1 where $d \le \text{radius}$, and 0 otherwise.
5. Zero the center: Explicitly set the center element (where $X=0, Y=0, Z=0$) to 0.

**Parameters:**
- `size`: Default 3.
- `radius`: Default 1.0.

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import numpy as np

def generate_3d_filled_circle_kernel(size: int = 3, radius: float = 1.0) -> np.ndarray:
    # 1. Create linear space from -(size//2) to size//2
    half = size // 2
    coords = np.arange(-half, half + 1)
    
    # 2. Generate 3D meshgrid of coordinates
    X, Y, Z = np.meshgrid(coords, coords, coords, indexing='ij')
    
    # 3. Compute radial distance d from center
    d = np.sqrt(X**2 + Y**2 + Z**2)
    
    # 4. Set kernel to 1 where d <= radius, else 0
    kernel = (d <= radius).astype(np.float32)
    
    # 5. Set center element to 0
    center_idx = half
    kernel[center_idx, center_idx, center_idx] = 0.0
    
    return kernel
```

**Edge Cases and Pitfalls:**
- **Center Exclusion:** The center voxel is explicitly forced to 0, even if the radius is large enough to include it. This is a specific requirement of this method [S1, unofficial].
- **Boundary Precision:** The condition $d \le \text{radius}$ includes voxels exactly on the boundary. Floating-point precision errors in distance calculation may cause slight variations in boundary voxels across implementations [S1, unofficial].
- **Odd vs Even Sizes:** The coordinate range `-(size//2)` to `size//2` assumes an odd-sized kernel for perfect symmetry around the origin. For even sizes, the center is offset, which may affect symmetry [S1, unofficial].

## Subtopics

### [Sphere Carving](2377-sphere-carving.md)
Sphere Carving removes solid voxels within a spherical radius from a center point in a voxel chunk, primarily used for destructible terrain scenarios. It differs from its parent [3D Filled Circle Kernel Generation](1520-3d-filled-circle-kernel-generation.md) by operating directly on the chunk data to modify state (removing voxels) rather than generating a static binary kernel, and from its sibling [3D Binary Sphere Kernel Generation](4654-3d-binary-sphere-kernel-generation.md) by returning a count of removed elements rather than a mask.

### [3D Binary Sphere Kernel Generation](4654-3d-binary-sphere-kernel-generation.md)
This method procedurally generates a 3D binary scalar field (density texture) representing a sphere centered at the origin of a voxel grid, specifically designed as input for GPU-based Marching Cubes surface extraction. Unlike static voxelization, this implementation features a time-varying radius that oscillates, creating a dynamic volume. It differs from its parent [3D Filled Circle Kernel Generation](1520-3d-filled-circle-kernel-generation.md) by explicitly targeting the `DensityTexture` property of a Marching Cubes component and utilizing a specific oscillating radius formula rather than a static threshold.

### 3D Gaussian Kernel Generation
Generates a 3D Gaussian kernel for smoothing operations, with the center element set to zero [S1, unofficial]. Steps: 1. Create linear space from `-(size//2)` to `size//2`. 2. Generate 3D meshgrid of coordinates. 3. Compute radial distance $d$ from center. 4. Compute Gaussian values: $\exp(-d^2 / (2\sigma^2))$. 5. Set center element to 0. 6. Normalize by sum of all elements [S1, unofficial]. Parameters: `size` (default 3), `sigma` (default 1.0) [S1, unofficial].

### Integer Brush Edit Plans
Deterministic generation of cell write sets for box and sphere shapes, emitting writes in canonical (z, y, x) order [S2, unofficial]. The sphere brush uses spall_core's fixed-point `contains_cell_centre` predicate over the shape's cell bounding box to determine included cells [S2, unofficial]. Pitfalls: The write set and order are byte-stable, guarded by pinned BLAKE3 digests and confirmed by a brute-force reference over a wider box [S2, unofficial]. Both brushes are verified to touch exactly eight bricks when centred on the corner where eight bricks meet [S2, unofficial]. A cell centre exactly on the sphere radius is included in the write set [S2, unofficial].

## Operating characteristics
**Time Complexity:** $O(N^3)$ where $N$ is the kernel size, as every voxel must be visited to compute distance and apply the threshold.
**Memory Complexity:** $O(N^3)$ to store the kernel array.
**Performance:** For small kernels (e.g., 3x3x3, 5x5x5), computation is negligible. For large kernels, the square root operation in distance calculation can be optimized by comparing squared distances ($d^2 \le \text{radius}^2$) to avoid expensive `sqrt` calls.
**Strengths:** Simple, deterministic, and easy to implement. Provides a clear binary representation of a spherical region.
**Limitations:** The explicit zeroing of the center voxel makes it unsuitable for applications requiring a continuous or non-zero center contribution (e.g., standard convolution kernels). The binary nature does not provide partial occupancy information.

## Background
This method is commonly used in voxel-based graphics and simulation for defining spherical regions of influence, such as in destructible environments or smoothing filters. The specific variant with center-zeroing is documented in the Plenoxels repository [S1, unofficial]. The deterministic integer brush variant is part of the spall project's fixture and brush system [S2, unofficial].

## Sources
[S1] DanJbk/Plenoxels, https://github.com/DanJbk/Plenoxels, fetched 2026-10-27 (unofficial)
[S2] T03 — Fixture worlds, rays, and integer brushes, https://github.com/deranjer/spall/pull/4, fetched 2026-10-25 (unofficial)
