# Splat Data Buffer Layout
Also called: SplatData Layout, Structure-of-Arrays Splat Storage
## Overview
This method defines the memory layout for storing 3D Gaussian splats in a Structure-of-Arrays (SoA) format, optimizing for batched access during rendering and optimization passes. It separates parameters into contiguous buffers to maximize memory bandwidth utilization and simplify SIMD vectorization, distinguishing it from Structure-of-Arrays (SoA) or Structure-of-Arrays (SoA) alternatives that might interleave data or use different packing schemes [S1, unofficial].
## How it works
**Inputs:** A collection of $N$ splat objects, each containing position, scale, rotation, opacity, and spherical harmonics coefficients.
**Outputs:** A set of contiguous `FloatBuffer` instances, one for each parameter type.

**Data Structures:**
The layout consists of separate contiguous buffers for each attribute:
1.  **Positions:** `FloatBuffer` of size $N \times 3$. Stores $(x, y, z)$ for each splat.
2.  **Scales:** `FloatBuffer` of size $N \times 3$. Stores $(s_x, s_y, s_z)$ for each splat.
3.  **Rotations:** `FloatBuffer` of size $N \times 4$. Stores quaternion components $(q_x, q_y, q_z, q_w)$ in scalar-last order.
4.  **Opacities:** `FloatBuffer` of size $N$. Stores a single opacity value per splat.
5.  **Spherical Harmonics:** `FloatBuffer` of size $N \times D \times 3$, where $D$ is the number of SH coefficients per channel.

**Algorithm:**
For each parameter type, iterate through the splat collection and write values into the corresponding buffer at index $i \times \text{stride} + \text{component}$, where $i$ is the splat index.

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def read_splat_buffers(splats: List[Splat]) -> Dict[str, List[float]]:
    """
    Extracts splat parameters into contiguous arrays (SoA layout).
    """
    n = len(splats)
    if n == 0:
        return {}

    # Pre-allocate buffers
    positions = [0.0] * (n * 3)
    scales = [0.0] * (n * 3)
    rotations = [0.0] * (n * 4)
    opacities = [0.0] * n
    # SH dimensions depend on the specific SH degree used; 
    # assuming D coefficients per channel, 3 channels (RGB)
    sh_dimensions = splats[0].sh_coefficients.shape[0] if splats[0].sh_coefficients else 0
    sh_buffer = [0.0] * (n * sh_dimensions * 3)

    for i, s in enumerate(splats):
        stride_pos = i * 3
        positions[stride_pos + 0] = s.get_position_x()
        positions[stride_pos + 1] = s.get_position_y()
        positions[stride_pos + 2] = s.get_position_z()

        stride_scale = i * 3
        scales[stride_scale + 0] = s.get_scale_x()
        scales[stride_scale + 1] = s.get_scale_y()
        scales[stride_scale + 2] = s.get_scale_z()

        stride_rot = i * 4
        rotations[stride_rot + 0] = s.get_rotation_x()
        rotations[stride_rot + 1] = s.get_rotation_y()
        rotations[stride_rot + 2] = s.get_rotation_z()
        rotations[stride_rot + 3] = s.get_rotation_w() # Scalar-last order

        opacities[i] = s.get_opacity()

        # Spherical Harmonics
        if sh_dimensions > 0:
            sh_coeffs = s.get_sh_coefficients() # Shape: (D, 3)
            for d in range(sh_dimensions):
                for c in range(3):
                    idx = (i * sh_dimensions + d) * 3 + c
                    sh_buffer[idx] = sh_coeffs[d][c]

    return {
        "positions": positions,
        "scales": scales,
        "rotations": rotations,
        "opacities": opacities,
        "spherical_harmonics": sh_buffer
    }
```

**Parameters:**
-   $N$: Number of splats.
-   $D$: Number of spherical harmonics coefficients per channel.

**Edge Cases:**
-   Empty splat collection: Returns empty buffers.
-   Null buffer input: In the reference implementation, a new direct buffer with native byte order is created if the result buffer is null [S1, unofficial].
-   Precision: Values are stored as 32-bit floats.

## Subtopics
## Operating characteristics
**Time Complexity:** $O(N)$ for extraction, where $N$ is the number of splats. The loop iterates once per splat, performing constant-time writes per attribute.
**Memory Complexity:** $O(N \times (3 + 3 + 4 + 1 + 3D))$ floats. The SoA layout ensures contiguous memory access, which is cache-friendly for rendering kernels that process all splats of a specific attribute type simultaneously.
**Strengths:** High throughput for batched operations; simplifies SIMD vectorization; avoids strided access patterns common in AoS (Array of Structures).
**Limitations:** Higher memory overhead if only a subset of attributes is needed for a specific pass, as all buffers are allocated.

## Background
This layout is implemented in the JSplat library, an unofficial Java implementation of 3D Gaussian Splatting techniques [S1, unofficial]. It follows the standard SoA pattern used in graphics programming to optimize memory access patterns for parallel processing.

## Sources
[S1] JSplat, https://github.com/javagl/JSplat, fetched 2026-10-30 (unofficial)
