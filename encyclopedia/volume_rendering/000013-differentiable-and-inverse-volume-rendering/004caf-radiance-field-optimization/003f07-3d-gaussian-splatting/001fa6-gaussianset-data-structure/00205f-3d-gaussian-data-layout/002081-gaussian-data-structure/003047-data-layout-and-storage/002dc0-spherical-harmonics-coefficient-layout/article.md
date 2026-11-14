# Spherical Harmonics Coefficient Layout

Also called: Per-Dimension SH Storage, SH Buffer Layout

## Overview
This method defines the memory layout for storing spherical harmonics (SH) coefficients for 3D Gaussians in a contiguous linear buffer [S1, unofficial]. It solves the problem of efficiently packing per-splat color data for GPU consumption by grouping coefficients by splat, then by coefficient index, and finally by spatial dimension (x, y, z) [S1, unofficial][5570-spherical-harmonics-coefficient-layout.md]. Builds on [Gaussian Data Structure](2273-gaussian-data-structure.md).

## How it works
**Inputs and Outputs:**
- Input: A collection of $N$ splats, each providing SH coefficients for a given degree.
- Output: A contiguous linear buffer (e.g., `FloatBuffer`) of size $N \times D \times 3$, where $D$ is the total number of SH coefficients for the given degree [S1, unofficial].

**Data Structures:**
- **SH Buffer:** A 1D array of floats with length $N \times D \times 3$ [S1, unofficial].
- **Indexing Formula:** The index for the $c$-th spatial component ($c \in \{0, 1, 2\}$) of the $d$-th coefficient ($d \in \{0, \dots, D-1\}$) of the $i$-th splat ($i \in \{0, \dots, N-1\}$) is:
  $$ idx = ((i \times D) + d) \times 3 + c $$
  This layout ensures that for a fixed splat and coefficient, the spatial components are contiguous in memory [S1, unofficial][5570-spherical-harmonics-coefficient-layout.md].

**Parameters:**
- $N$: Number of splats.
- $D$ (`shDimensions`): Total number of coefficients, calculated via `Splats.dimensionsForDegree(shDegree)` [S1, unofficial].

**Steps:**
1. Calculate $D$ using the degree-to-dimension mapping function [S1, unofficial].
2. Allocate a buffer of size $N \times D \times 3$ [S1, unofficial].
3. Iterate over each splat index $i$ from $0$ to $N-1$ [S1, unofficial].
4. For each splat, iterate over each coefficient index $d$ from $0$ to $D-1$ [S1, unofficial].
5. For each coefficient, write the three spatial components to buffer indices $((i \times D) + d) \times 3 + 0$, $+1$, and $+2$ [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def write_sh_coefficients(splats, sh_degree):
    """
    Writes SH coefficients into a contiguous linear buffer.
    
    Args:
        splats: List of N splat objects, each having a method get_sh(d) 
                that returns a tuple/list of 3 floats (x, y, z) for coefficient d.
        sh_degree: Integer, the degree of the spherical harmonics.
        
    Returns:
        A list of floats representing the SH buffer.
    """
    # Calculate total number of coefficients per splat
    # Implementation detail from source: Splats.dimensionsForDegree(shDegree)
    sh_dimensions = dimensions_for_degree(sh_degree)
    
    # Initialize buffer
    buffer_size = len(splats) * sh_dimensions * 3
    buffer = [0.0] * buffer_size
    
    # Iterate over each splat
    for i, splat in enumerate(splats):
        # Iterate over each coefficient index
        for d in range(sh_dimensions):
            # Get the x, y, z components for this coefficient
            x, y, z = splat.get_sh(d)
            
            # Calculate base index for this coefficient
            base_idx = (i * sh_dimensions + d) * 3
            
            # Write x, y, z to contiguous positions
            buffer[base_idx + 0] = x
            buffer[base_idx + 1] = y
            buffer[base_idx + 2] = z
            
    return buffer
```

**Edge Cases and Pitfalls:**
- **Integer Overflow:** For very large $N$, ensure the buffer index calculation does not overflow 32-bit integers [G].
- **Degree Validation:** Ensure $L$ is non-negative before calculating $D$ [G].

## Subtopics

### Spherical Harmonics Texture Management
Instead of a linear buffer, SH coefficients are stored in separate GPU textures keyed by block identifiers (`this.shTextures[key]`) [S2, unofficial]. The system iterates over all `SH_KEYS`, calling `ensureShTexture` to either reuse an existing texture, mark it for update if `needsUpdate` is true, or create a new texture from `this.extra[key]` if data exists [S2, unofficial]. If a texture already exists and `this.needsUpdate` is true, `current.needsUpdate` is set to true to signal the GPU driver to refresh the texture data [S2, unofficial].

### INT8 Spherical Harmonics
This variant uses INT8 quantization for spherical harmonics coefficients to reduce memory traffic [S3, unofficial].

## Operating characteristics
- **Time Complexity:** Writing coefficients is $O(N \times D)$ [G]. Reading coefficients in a shader is $O(1)$ per coefficient with direct indexing [G].
- **Memory Complexity:** $O(N \times D \times 3)$ floats [S1, unofficial].
- **Strengths:** Direct indexing without texture sampling overhead [S1, unofficial].
- **Limitations:** Memory usage scales linearly with $N$ and quadratically with SH degree [G].

## Background
The linear buffer layout is a standard implementation detail in 3D Gaussian Splatting frameworks for CPU-side data preparation before GPU upload [S1, unofficial]. Variants like texture-based management and INT8 quantization have been introduced to address specific performance bottlenecks in large-scale or streaming scenarios [S2, unofficial][S3, unofficial].

## Sources
[S1] JSplat, https://github.com/javagl/JSplat, fetched 2026-10-30 (unofficial)
[S2] WilliamLiu-1997/Gaussian-Splat-Lite, https://github.com/WilliamLiu-1997/Gaussian-Splat-Lite, fetched 2026-10-27 (unofficial)
[S3] RoofGS: Roofline-Guided End-to-End Acceleration of 3D Gaussian Splatting, https://arxiv.org/abs/2608.15785, fetched 2026-10-29 (unofficial)
