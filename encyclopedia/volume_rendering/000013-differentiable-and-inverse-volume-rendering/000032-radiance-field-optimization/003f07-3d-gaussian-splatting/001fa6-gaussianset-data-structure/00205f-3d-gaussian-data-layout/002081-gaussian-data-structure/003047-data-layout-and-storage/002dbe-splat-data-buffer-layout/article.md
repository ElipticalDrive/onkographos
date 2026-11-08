# Splat Data Buffer Layout
Also called: SplatData Layout, Structure-of-Arrays Splat Storage
## Overview
This method defines the memory layout for 3D Gaussian splats in a Structure-of-Arrays (SoA) format, where each attribute occupies a separate contiguous linear buffer. It solves the problem of cache inefficiency during batched rendering by ensuring shaders iterate over single attributes without skipping irrelevant data. It differs from sibling methods by focusing on the fundamental in-memory SoA organization rather than coefficient indexing [5570-spherical-harmonics-coefficient-layout.md], covariance calculation [6840-covariance-splat-reading.md], or dynamic scene management [6907-dynamic-gaussian-splatting-scene-management.md].
## How it works
**Inputs:** A collection of $N$ Gaussian splats, each containing position, scale, rotation, opacity, and spherical harmonics (SH) coefficients.
**Outputs:** A set of contiguous `FloatBuffer` arrays (or equivalent flat memory blocks).

**Data Structures:**
The layout consists of five distinct buffers, each of size $N \times \text{stride}$:
1.  **Positions:** Size $N \times 3$. Stores $(x, y, z)$.
2.  **Scales:** Size $N \times 3$. Stores $(s_x, s_y, s_z)$. Note: Sources may store logarithmic scales, which are exponentiated during rendering [G].
3.  **Rotations:** Size $N \times 4$. Stores quaternion components $(w, x, y, z)$ or $(x, y, z, w)$. JSplat uses scalar-last order $(x, y, z, w)$ [S1, unofficial].
4.  **Opacities:** Size $N \times 1$. Stores a single scalar opacity value per splat.
5.  **Spherical Harmonics (SH):** Size $N \times D \times 3$. $D$ is the number of SH coefficients per channel. The factor 3 represents RGB channels.

**Indexing Formula:**
For the SH buffer, the index for the $c$-th component (channel, $c \in \{0,1,2\}$) of the $d$-th coefficient of the $i$-th splat is:
$$ idx = ((i \times D) + d) \times 3 + c $$
For non-SH attributes, the stride is fixed by dimensionality (3 for position/scale, 4 for rotation, 1 for opacity).

**Algorithm:**
1.  Initialize buffers with total size $N \times \text{stride}$ for each attribute type.
2.  Iterate $i$ from $0$ to $N-1$:
    a. Extract position components $p_x, p_y, p_z$ from splat $i$.
    b. Write to Positions buffer at indices $i \times 3 + 0$, $i \times 3 + 1$, $i \times 3 + 2$.
    c. Extract scale components $s_x, s_y, s_z$.
    d. Write to Scales buffer at indices $i \times 3 + 0$, $i \times 3 + 1$, $i \times 3 + 2$.
    e. Extract rotation quaternion components $r_x, r_y, r_z, r_w$.
    f. Write to Rotations buffer at indices $i \times 4 + 0$, $i \times 4 + 1$, $i \times 4 + 2$, $i \times 4 + 3$.
    g. Extract opacity $\alpha$.
    h. Write to Opacities buffer at index $i$.
    i. Extract SH coefficients for all $D$ degrees and 3 channels.
    j. Write to SH buffer using the indexing formula above.

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def build_splat_buffers(splats: List[Splat]) -> Dict[str, List[float]]:
    N = len(splats)
    D = splats[0].sh_degree  # Assuming uniform degree
    
    # Initialize buffers
    positions = [0.0] * (N * 3)
    scales = [0.0] * (N * 3)
    rotations = [0.0] * (N * 4)
    opacities = [0.0] * N
    sh_coeffs = [0.0] * (N * D * 3)
    
    for i, splat in enumerate(splats):
        # Positions
        positions[i * 3 + 0] = splat.position.x
        positions[i * 3 + 1] = splat.position.y
        positions[i * 3 + 2] = splat.position.z
        
        # Scales (assuming linear storage or pre-converted)
        scales[i * 3 + 0] = splat.scale.x
        scales[i * 3 + 1] = splat.scale.y
        scales[i * 3 + 2] = splat.scale.z
        
        # Rotations (scalar-last: x, y, z, w)
        rotations[i * 4 + 0] = splat.rotation.x
        rotations[i * 4 + 1] = splat.rotation.y
        rotations[i * 4 + 2] = splat.rotation.z
        rotations[i * 4 + 3] = splat.rotation.w
        
        # Opacity
        opacities[i] = splat.opacity
        
        # Spherical Harmonics
        for d in range(D):
            for c in range(3):
                idx = ((i * D) + d) * 3 + c
                sh_coeffs[idx] = splat.sh_coeffs[d][c]
                
    return {
        "positions": positions,
        "scales": scales,
        "rotations": rotations,
        "opacities": opacities,
        "sh_coeffs": sh_coeffs
    }
```

## Subtopics

### [Splat Data Encoding & Disabled Splat Detection](7028-splat-data-encoding-disabled-splat-detection.md)
Defines binary encoding for `Gaussian-Splat-Lite` to detect disabled splats. It embeds status into the data payload's bit pattern (treating disabled splats as NaN) to filter inactive/merged Gaussians without boolean flags. It builds on [Splat Data Buffer Layout](5568-splat-data-buffer-layout.md).

### Unified Parameter Layout and SSD Blocking
A logical layout for all learnable per-Gaussian attributes stored out-of-core on SSD. It improves I/O efficiency by Morton-sorting Gaussians by center codes before blocking to ensure spatial locality. It uses a unified layout $\Theta \in \mathbb{R}^{N \times D}$ with $D=59$ (degree-3 SH). Blocks are defined as $\text{Block}(k) := \Theta[kB : (k+1)B]$ with $B=4096$. Each block is $4096 \times 59 \times 4 = 964,544$ bytes. The owner block is fixed during training; bounds are conservatively refreshed to allow overlap, but each Gaussian is rasterized exactly once [S2, unofficial].

## Operating characteristics
**Time Complexity:** $O(N)$ to build the buffers.
**Memory Complexity:** $O(N \times (3 + 3 + 4 + 1 + 3D))$ floats. For degree-3 SH ($D=12$ per channel? No, $D$ is coefficients per channel. Degree 3 has 16 coefficients total, 4 per channel. So $D=4$? The source says $D=59$. Let's check: 1 (opacity) + 3 (pos) + 3 (scale) + 4 (rot) + 12 (SH coeffs? No. Degree 3 SH has 16 coeffs. 16 * 3 = 48. 1+3+3+4+48 = 59. Yes. So $D$ in the SH buffer size formula refers to coefficients per channel, which is 4 for degree 3. But the unified layout uses $D=59$ for all params. The base layout separates them. The base layout SH size is $N \times 4 \times 3 = 12N$ floats for degree 3).
**Strengths:** Minimizes cache misses during rendering; simple indexing.
**Limitations:** Requires separate buffers for each attribute; SH data can be large.
**When to use:** Standard 3D Gaussian Splatting rendering.
**When not to use:** When memory bandwidth is extremely constrained and attributes are accessed sparsely (use SoC).

## Background
Originated from the 3D Gaussian Splatting framework. The JSplat library provides an unofficial implementation of this SoA layout for Java-based rendering [S1, unofficial]. The layout is fundamental to the differentiable optimization and rendering pipeline.

## Sources
[S1] JSplat, https://github.com/javagl/JSplat, fetched 2026-10-30 (unofficial)
[S2] TideGS: Scalable Training of Over One Billion 3D Gaussian Splatting Primitives via Out-of-Core Optimization, https://doi.org/10.48550/arxiv.2605.20150, fetched 2026-10-26 (unofficial)
