# fBm Noise Generation
Also called: Water Vapour Emulation (fBm Noise Generation), fBm Noise Generation for Cloud Density, fBm density simulation, fractal Brownian motion noise, uniform noise cube

## Overview
This method simulates the irregular density of water droplets in clouds by generating fractal Brownian motion (fBm) noise from a pre-computed 3D noise cube [S1]. It transforms uniform random noise into a Gaussian-like distribution suitable for atmospheric vapour simulation, serving as the density field input for ray-marching algorithms [S1]. It differs from procedural noise generation (e.g., Perlin or Simplex noise computed on-the-fly) by relying on a static, pre-calculated texture lookup, which trades memory bandwidth for computational simplicity and deterministic results [S1].

## How it works
**Inputs and Outputs**
*   **Input**: A 3D texture (volume) containing pre-computed uniform random noise values.
*   **Input**: Camera ray origin $\mathbf{o}$ and direction $\mathbf{d}$ (for spatial sampling coordinates).
*   **Input**: Number of octaves $N$, scale factor $w$, and sampling factor $s$.
*   **Output**: A scalar density value $\rho(\mathbf{p})$ at position $\mathbf{p}$, derived from the fBm sum.

**Data Structures**
*   **Noise Cube**: A 3D array of dimensions $64 \times 64 \times 64$ [S1].
*   **Data Type**: Single-precision floating-point values (32-bit) [S1].
*   **Content**: Uniform random noise values in the range $[0, 1]$ or $[-1, 1]$ (typically $[0,1]$ for standard texture sampling, mapped appropriately) [S1].
*   **Storage**: Transferred from host CPU to GPU memory [S1].

**Parameters**
*   **Number of octaves ($N$)**: 5 [S1]. Increasing beyond 5 does not improve vapour deformation shape and decreases frame performance [S1].
*   **Octave scale factor ($w$)**: Attenuation factor for amplitude. The source states $w < 1$ but does not provide the exact numerical value used in the implementation [S1].
*   **Noise sampling factor ($s$)**: Frequency multiplier for each octave. The source states $s > 1$ but does not provide the exact numerical value used in the implementation [S1].
*   **Note**: The source mentions choosing specific values for $w$ and $s$ but the text snippet omits the exact numbers, stating only "where we choose and ." [S1].

**Algorithm Steps**
1.  **Pre-computation (Host CPU)**:
    *   Generate a $64 \times 64 \times 64$ grid of single-precision floats.
    *   Fill each voxel with a uniform random noise value.
    *   Transfer this 3D cube to GPU memory as a texture [S1].

2.  **Sampling (GPU Fragment Shader)**:
    *   For each pixel, compute the ray position $\mathbf{p}$ along the ray $r(t) = \mathbf{o} + t\mathbf{d}$ [S1].
    *   Initialize `totalNoise = 0.0` and `currentScale = 1.0` [G].
    *   Iterate through octaves $i$ from $0$ to $N-1$ (where $N=5$) [S1]:
        *   Sample the noise cube at position $\mathbf{p} \cdot s^i$ [S1].
        *   Retrieve the uniform noise value $U_i$ from the texture [S1].
        *   Accumulate the weighted noise: `totalNoise += w^i * U_i` [S1].
        *   Update scale for next octave: `currentScale *= s` [G].
    *   The implementation uses an **unrolled** version of this summation in the fragment shader to improve performance [S1].

**Formula**
The fBm noise value at position $\mathbf{x}$ is defined as [S1]:
$$ fBm(\mathbf{x}) = \sum_{i=0}^{N-1} w^i \cdot Noise(s^i \cdot \mathbf{x}) $$
Where:
*   $N = 5$ is the number of octaves [S1].
*   $w$ is the octave scale factor (amplitude decay), with $w < 1$ [S1].
*   $s$ is the noise sampling factor (frequency increase), with $s > 1$ [S1].
*   $Noise(\cdot)$ is the lookup function returning the uniform random value from the pre-computed 3D cube [S1].
*   $\mathbf{x}$ is the 3D spatial coordinate [S1].

**Edge Cases and Pitfalls**
*   **Precision**: Single-precision floats are used for the noise cube [S1].
*   **Performance**: Increasing octaves beyond 5 yields diminishing returns for vapour shape while reducing frame rate [S1].
*   **Boundary Conditions**: The noise cube is fixed at $64^3$. Sampling outside this range requires wrapping or clamping, though the source does not specify the wrapping mode [S1].
*   **Determinism**: Since the noise is pre-computed, the result is deterministic and identical across frames for the same input coordinates, unlike procedural noise which might vary if not seeded consistently [S1].

**Pseudocode**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
# Pre-computation (Host CPU)
def generate_noise_cube(size=64):
    """
    Generates a 3D array of uniform random noise.
    Returns a flat list or 3D array of single-precision floats.
    """
    import random
    # Implementation detail: Generate size*size*size random floats
    noise_cube = []
    for x in range(size):
        for y in range(size):
            for z in range(size):
                noise_cube.append(random.uniform(0.0, 1.0)) # [G] Assuming [0,1] range
    return noise_cube

# GPU Shader Logic (Unrolled for N=5)
def sample_fbm(position, noise_texture, w, s):
    """
    Samples the pre-computed noise cube using fBm summation.
    position: vec3, the 3D coordinate to sample.
    noise_texture: 3D texture object containing the uniform noise cube.
    w: float, octave scale factor (amplitude).
    s: float, noise sampling factor (frequency).
    """
    # Unrolled loop for 5 octaves as per source [S1]
    # Octave 0
    p0 = position * (s ** 0)
    n0 = texture3D(noise_texture, p0).r # [G] Assuming red channel or single value
    
    # Octave 1
    p1 = position * (s ** 1)
    n1 = texture3D(noise_texture, p1).r
    
    # Octave 2
    p2 = position * (s ** 2)
    n2 = texture3D(noise_texture, p2).r
    
    # Octave 3
    p3 = position * (s ** 3)
    n3 = texture3D(noise_texture, p3).r
    
    # Octave 4
    p4 = position * (s ** 4)
    n4 = texture3D(noise_texture, p4).r
    
    # Summation
    # Note: Exact values for w and s are not provided in source [S1]
    # Assuming w < 1 and s > 1
    fbm = (w**0 * n0) + (w**1 * n1) + (w**2 * n2) + (w**3 * n3) + (w**4 * n4)
    
    return fbm
```

## Subtopics

### Procedural fBm (Perlin/Simplex)
Generates noise on-the-fly using gradient vectors and interpolation. More flexible but computationally heavier than pre-computed cubes. Choose when memory is constrained or dynamic noise evolution is needed. [Link to sibling if available, otherwise note as alternative].

### Voxel-based Density
Uses explicit voxel grids rather than noise functions. Better for sharp, structured clouds but requires more memory for high resolution. Choose for non-stochastic, geometric cloud shapes.

## Operating characteristics
*   **Time Complexity**: $O(N)$ per sample, where $N$ is the number of octaves (fixed at 5) [S1]. The unrolled implementation ensures constant time overhead per sample [S1].
*   **Memory Complexity**: $O(V)$, where $V$ is the volume of the noise cube. For $64^3$ single-precision floats, this is $64 \times 64 \times 64 \times 4$ bytes $\approx 1$ MB [S1].
*   **Measured Performance**: The source states that increasing octaves above 5 decreases frame performance [S1]. Pre-calculating the noise on the host and transferring it to the GPU achieves higher frames-per-second compared to procedural generation [S1].
*   **Strengths**: Fast sampling (texture lookup), deterministic results, low computational cost per sample [S1].
*   **Limitations**: Fixed resolution ($64^3$) limits detail at high zoom levels; requires memory transfer; less flexible than procedural noise [S1].
*   **When to use**: Real-time volumetric cloud rendering where performance is critical and static noise patterns are acceptable [S1].
*   **When not to use**: When dynamic noise evolution or infinite resolution is required [S1].

## Background
This method was described in the context of efficient algorithms for real-time GPU volumetric cloud rendering [S1]. It builds on the concept of fractal Brownian motion (fBm) as a fractal sum of noise functions, referencing Iñigo Quilez’s work on noise generation [S1]. The specific implementation uses uniform random noise instead of Perlin noise for the base layer, transforming it into a Gaussian-like distribution via octave accumulation [S1]. The technique is applied to simulate water vapour density in clouds, leveraging the stochastic properties of noise to emulate natural textures [S1].

## Sources
[S1] Efficient Algorithms for Real-Time GPU Volumetric Cloud Rendering with Enhanced Geometry, https://doi.org/10.3390/sym10040125, fetched 2026-10-26
