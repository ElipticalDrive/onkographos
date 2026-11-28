# Spherical Harmonics Coefficient Layout

Also called: Per-Dimension SH Storage, SH Buffer Layout

## Overview
This layout defines the memory organization for storing spherical harmonic (SH) coefficients within the Structure-of-Arrays (SoA) paradigm used in 3D Gaussian Splatting. It solves the problem of efficiently mapping per-Gaussian color data (represented by SH coefficients) into contiguous linear buffers for GPU transfer and rendering. It differs from interleaved layouts by grouping coefficients by spatial dimension (RGB) after grouping by splat and degree, optimizing for vectorized processing of the RGB triplet during the final color computation step [S1, unofficial].

## How it works
**Inputs and Outputs:**
- Input: A collection of $N$ Gaussian splats, each having a set of SH coefficients up to a specific degree.
- Output: A contiguous linear buffer (e.g., `FloatBuffer`) of size $N \times D \times 3$, where $D$ is the total number of SH coefficients for the chosen degree.

**Data Structures:**
The layout uses a single contiguous array of 32-bit floats. The indexing formula for the coefficient at splat index $i$, coefficient index $d$, and color channel $c$ (where $c \in \{0, 1, 2\}$ for R, G, B) is:
$$ idx = ((i \times D) + d) \times 3 + c $$
Here, $D$ is the total number of SH coefficients for the given degree, calculated via `Splats.dimensionsForDegree(shDegree)` [S1, unofficial].

**Steps:**
1. Determine the total number of SH coefficients $D$ for the selected degree.
2. Allocate a linear buffer of size $N \times D \times 3$.
3. Iterate over each splat index $i$ from $0$ to $N-1$.
4. For each splat, iterate over each coefficient index $d$ from $0$ to $D-1$.
5. For each coefficient, write the three components (X, Y, Z corresponding to R, G, B) to the buffer at indices:
   - $((i \times D) + d) \times 3 + 0$ for the first component (e.g., Red) [S1, unofficial].
   - $((i \times D) + d) \times 3 + 1$ for the second component (e.g., Green) [S1, unofficial].
   - $((i \times D) + d) \times 3 + 2$ for the third component (e.g., Blue) [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def write_sh_coefficients(splats: List[Splat], sh_degree: int) -> List[float]:
    """
    Writes SH coefficients into a flat buffer following the per-dimension layout.
    
    Args:
        splats: List of N Splat objects.
        sh_degree: The degree of spherical harmonics used.
        
    Returns:
        A flat list of floats representing the buffer.
    """
    # D is the total number of SH coefficients for the given degree
    # Formula: D = (sh_degree + 1)^2
    D = (sh_degree + 1) ** 2
    
    N = len(splats)
    buffer_size = N * D * 3
    buffer = [0.0] * buffer_size
    
    for i in range(N):
        splat = splats[i]
        for d in range(D):
            # Get the three components for coefficient d
            # Note: In standard SH, these correspond to R, G, B values for that coefficient
            sh_x = splat.get_sh_x(d)
            sh_y = splat.get_sh_y(d)
            sh_z = splat.get_sh_z(d)
            
            # Calculate base index for this coefficient
            base_idx = (i * D + d) * 3
            
            # Write components to buffer
            buffer[base_idx + 0] = sh_x
            buffer[base_idx + 1] = sh_y
            buffer[base_idx + 2] = sh_z
            
    return buffer
```

**Parameters:**
- `shDegree`: The degree of spherical harmonics. Typical values are 0, 1, 2, or 3. Degree 0 has 1 coefficient, Degree 1 has 4, Degree 2 has 9, Degree 3 has 16 [S1, unofficial].
- `N`: Number of splats.

**Edge Cases and Pitfalls:**
- **Degree Mismatch:** Ensure $D$ is calculated consistently across all splats. If splats have different degrees, this layout requires padding or separate buffers.
- **Precision:** Coefficients are stored as 32-bit floats. Quantization methods (e.g., INT8) may be applied to reduce memory traffic [S3, unofficial], but this layout assumes float storage by default.
- **Boundary Conditions:** The buffer size must be exactly $N \times D \times 3$. Under-allocation causes out-of-bounds errors; over-allocation wastes memory.

## Subtopics

### [SH Coefficient Interleaving](15882-sh-coefficient-interleaving.md)
SH Coefficient Interleaving packs spherical harmonic (SH) coefficients into a single contiguous array by grouping coefficients by channel rather than by degree. It solves the problem of efficient memory access during rendering by ensuring that all color channels for a specific SH degree are stored adjacently. It differs from non-interleaved layouts by optimizing for vectorized processing of the RGB triplet during the final color computation step.

### Spherical Harmonics Texture Management
This variant manages SH coefficients by storing them in separate textures keyed by block identifiers (e.g., `this.shTextures[key]`) [S2, unofficial]. It iterates over all SH keys, calling `ensureShTexture` to either reuse an existing texture, mark it for update if `needsUpdate` is true, or create a new texture from `this.extra[key]` if data exists [S2, unofficial]. If a texture already exists and `this.needsUpdate` is true, it sets `current.needsUpdate = true` to signal the GPU driver to refresh the texture data [S2, unofficial]. This approach is used in streaming contexts where SH data is loaded dynamically [S2, unofficial].

### INT8 Spherical Harmonics
This variant uses INT8 quantization for spherical harmonics coefficients to reduce memory traffic [S3, unofficial]. It is part of hardware-oriented optimizations like RoofGS, which aims to accelerate rendering through reduced data size and efficient use of Tensor Cores [S3, unofficial].

## Operating characteristics
- **Time Complexity:** Writing coefficients is $O(N \times D)$, where $N$ is the number of splats and $D$ is the number of SH coefficients per splat. Reading for rendering is also $O(N \times D)$ if all coefficients are accessed.
- **Memory Complexity:** The layout requires $N \times D \times 3 \times 4$ bytes of memory (assuming 32-bit floats). For $N=10^6$ and Degree 3 ($D=16$), this is approximately 192 MB.
- **Performance:** The contiguous memory layout facilitates efficient bulk transfers to the GPU. However, the non-interleaved nature (grouping by splat then degree then channel) may require specific indexing logic in shaders compared to interleaved layouts.
- **Strengths:** Simple indexing formula, easy to implement, compatible with standard SoA buffers.
- **Limitations:** Not optimized for vectorized RGB access during rendering without specific shader logic. High memory usage for high-degree SH.

## Background
This layout is a fundamental component of the 3D Gaussian Splatting rendering pipeline, specifically within the Gaussian Splatting family of techniques. It was introduced in the original 3D Gaussian Splatting paper [Kerbl et al., 2023] and is widely adopted in implementations such as JSplat [S1, unofficial] and Gaussian-Splat-Lite [S2, unofficial]. The layout supports the dynamic scene management and incremental updates required for real-time rendering of large-scale scenes.

## Sources
[S1] JSplat, https://github.com/javagl/JSplat, fetched 2026-10-30 (unofficial)
[S2] WilliamLiu-1997/Gaussian-Splat-Lite, https://github.com/WilliamLiu-1997/Gaussian-Splat-Lite, fetched 2026-10-27 (unofficial)
[S3] RoofGS: Roofline-Guided End-to-End Acceleration of 3D Gaussian Splatting, https://arxiv.org/abs/2608.15785, fetched 2026-10-29 (unofficial)
