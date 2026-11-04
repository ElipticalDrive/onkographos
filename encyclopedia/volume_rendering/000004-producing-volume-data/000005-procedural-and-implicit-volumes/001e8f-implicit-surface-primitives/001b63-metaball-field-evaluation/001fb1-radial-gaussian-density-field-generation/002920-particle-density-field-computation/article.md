# Particle Density Field Computation

Also called: Particle Power Summation, Particle-to-Grid Transfer

## Overview
Particle Density Field Computation generates a 3D scalar density field by evaluating a radial kernel function for every voxel in a 3D grid, summing contributions from a set of particles. It solves the problem of converting discrete particle data into a continuous volumetric representation suitable for ray marching or marching cubes. Unlike [Radial Gaussian Density Field Generation](2081-radial-gaussian-density-field-generation.md), which uses smooth Gaussian falloffs, this method employs a specific polynomial kernel with a sharp cutoff at a normalized squared distance of 0.5, resulting in a compact support that strictly excludes distant particles [S1, unofficial].

## How it works
**Inputs and Outputs:**
*   **Input:** A buffer of particles, where each particle is defined by a position $\mathbf{p} = (x, y, z)$ and a radius $r$. A 3D grid of resolution $W \times H \times D$ is defined by a voxel spacing `textureSizeScale`.
*   **Output:** A 3D scalar field (density) stored in a buffer, where each voxel contains the accumulated density value.

**Data Structures:**
*   `posBuffer`: A structured buffer of `float4` values. Each element stores a particle as $(x, y, z, r)$, where the first three components are the world position and the fourth component `w` is the particle radius $r$ [S1, unofficial].
*   `pixels`: A structured buffer of `float4` values representing the output grid. Each element stores the computed density in the first three components $(d, d, d)$ and a constant alpha of $1.0$ [S1, unofficial].
*   `g_Params`: A `float4` constant buffer. `g_Params.z` holds `textureSizeScale` (voxel spacing). `g_Params.y` holds the grid dimension in Y (used for linearization). `g_Params.x` and `w` are unused in the core logic described [S1, unofficial].

**Algorithm Steps:**
1.  **Dispatch:** Launch a compute shader with thread groups of size $8 \times 8 \times 8$. Each thread is responsible for computing the density of one voxel [S1, unofficial].
2.  **Voxel Position:** Calculate the world position $\mathbf{t}$ of the current voxel using the thread ID $(id.x, id.y, id.z)$ and the voxel spacing `textureSizeScale`:
    $$ \mathbf{t} = (id.x \cdot \text{textureSizeScale}, \ id.y \cdot \text{textureSizeScale}, \ id.z \cdot \text{textureSizeScale}) $$
3.  **Accumulator Initialization:** Initialize a scalar accumulator $p$ to $0.0$ [S1, unofficial].
4.  **Particle Iteration:** Iterate through all $N$ particles in `posBuffer`. For each particle $i$ with position $\mathbf{p}_i$ and radius $r_i$:
    a. Calculate the squared Euclidean distance between the voxel $\mathbf{t}$ and the particle center $\mathbf{p}_i$:
       $$ \|\mathbf{p}_i - \mathbf{t}\|^2 = (\mathbf{p}_i.x - \mathbf{t}.x)^2 + (\mathbf{p}_i.y - \mathbf{t}.y)^2 + (\mathbf{p}_i.z - \mathbf{t}.z)^2 $$
    b. Compute the normalized squared distance $m^2$:
       $$ m^2 = \frac{\|\mathbf{p}_i - \mathbf{t}\|^2}{r_i^2} $$
    c. Evaluate the kernel function $f(m^2)$:
       $$ f(m^2) = \begin{cases} 0.25 - m^2 + (m^2)^2 & \text{if } m^2 < 0.5 \\ 0.0 & \text{if } m^2 \ge 0.5 \end{cases} $$
    d. Add the result to the accumulator: $p \leftarrow p + f(m^2)$ [S1, unofficial].
5.  **Write Output:** Store the accumulated density $p$ into the output buffer at the linearized index corresponding to the voxel $(id.x, id.y, id.z)$:
    $$ \text{pixels}[idx] = (p, p, p, 1.0) $$

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
# Constants
NUM_PARTICLES = len(posBuffer)
TEXTURE_SIZE_SCALE = g_Params.z
GRID_DIM_Y = g_Params.y

# Kernel function
def get_power(test_pos, particle):
    # particle is (x, y, z, radius)
    dx = particle.x - test_pos.x
    dy = particle.y - test_pos.y
    dz = particle.z - test_pos.z
    
    # Squared distance
    m_sqr = dx*dx + dy*dy + dz*dz
    
    # Normalize by radius squared (particle.w is radius)
    radius_sq = particle.w * particle.w
    m_sqr = m_sqr / radius_sq
    
    # Apply kernel with sharp cutoff
    if m_sqr >= 0.5:
        return 0.0
    
    # Polynomial kernel
    return 0.25 - m_sqr + (m_sqr * m_sqr)

# Compute Shader Main
def compute_voxel(thread_id):
    # thread_id is (x, y, z)
    
    # 1. Calculate world position of the voxel
    test_pos_x = thread_id.x * TEXTURE_SIZE_SCALE
    test_pos_y = thread_id.y * TEXTURE_SIZE_SCALE
    test_pos_z = thread_id.z * TEXTURE_SIZE_SCALE
    test_pos = (test_pos_x, test_pos_y, test_pos_z)
    
    # 2. Initialize accumulator
    density = 0.0
    
    # 3. Iterate over all particles
    for i in range(NUM_PARTICLES):
        particle = posBuffer[i]
        contribution = get_power(test_pos, particle)
        density += contribution
        
    # 4. Linearize index for output buffer
    # Assuming row-major order: idx = z * (W * H) + y * W + x
    # Note: Source uses g_Params.y for Y dimension, but W and H are not explicitly 
    # provided in g_Params. We assume standard linearization or that W/H are known constants.
    # [G] Standard linearization requires width and height. 
    # If grid is cubic or dimensions are known, idx = z * W * H + y * W + x.
    # The source code snippet implies a linear index calculation exists.
    idx = thread_id.z * (GRID_DIM_X * GRID_DIM_Y) + thread_id.y * GRID_DIM_X + thread_id.x
    
    # 5. Write result
    pixels[idx] = (density, density, density, 1.0)
```

**Parameters:**
*   `textureSizeScale`: The physical spacing between adjacent grid voxels [S1, unofficial].
*   `numParticles`: The total count of particles in `posBuffer` [S1, unofficial].
*   Kernel Cutoff: The normalized squared distance threshold is hardcoded to $0.5$ [S1, unofficial].

**Edge Cases and Pitfalls:**
*   **Sharp Cutoff:** The kernel function returns exactly $0.0$ for $m^2 \ge 0.5$. This creates a sharp boundary rather than a smooth falloff, which may lead to aliasing or visible artifacts if the grid resolution is insufficient relative to the particle radii [S1, unofficial].
*   **Precision:** Floating-point arithmetic is used for the accumulation. For very large numbers of particles or very small radii, precision loss may occur, though the specific kernel limits the range of values added [S1, unofficial].

## Subtopics

### Quantization for Exact Arithmetic
A technique to prevent higher-order rounding error propagation during the recursive computation of polynomial coefficients used in particle-wise field approximation [S2]. It converts floating-point quantities to fixed-point integers to ensure exact arithmetic [S2].
*   **Definition:** Shifts numbers encoding contribution approximations into integer multiples of quantum values. Knots $(t_k, \hat{a}_{k0}, \dots, \hat{a}_{kD})$ and localized coefficients $a_{kd}$ are quantized such that $t_k \approx \bar{t}_k \tau$ and $\hat{a}_{kd} \approx \bar{\hat{a}}_{kd} \zeta_d$, $a_{kd} \approx \bar{a}_{kd} \zeta_d$ for integers $\bar{t}_k, \bar{\hat{a}}_{kd}, \bar{a}_{kd}$ [S2].
*   **Formula:** The update rule becomes integer-only:
    $$ \bar{a}_{kd} = \bar{a}_{kd} + \sum_{j=d}^D \binom{j}{d} \bar{a}_{(k-1)j} (\bar{t}_k - \bar{t}_{k-1})^{j-d} $$
    provided $\zeta_d = \frac{\zeta}{\tau^d}$ [S2].
*   **Parameters:** $\tau$ is the length quantum value (resolution of knot positions). $\zeta$ is the field value quantum value (resolution of coefficients, where $\zeta_0 = \zeta$) [S2].
*   **Pitfall:** Direct application of the recursive coefficient update using floating-point numbers causes rounding errors to propagate at higher order from front to back along the viewing ray, leading to unstable results and 'sprinkling' artifacts [S2].

## Operating characteristics
*   **Time Complexity:** $O(N \cdot V)$, where $N$ is the number of particles and $V$ is the number of voxels in the grid. Each voxel iterates over all particles.
*   **Memory Complexity:** $O(N + V)$ for storing the particle buffer and the output density grid.
*   **Performance:** The method is highly parallelizable and well-suited for GPU compute shaders, as each voxel calculation is independent [S1, unofficial].
*   **Strengths:** Simple implementation; compact support (sharp cutoff) reduces the number of effective contributions per voxel if spatial hashing or bounding volumes are added (though the basic algorithm iterates all); deterministic results.
*   **Limitations:** The $O(N \cdot V)$ complexity becomes prohibitive for large $N$ and high-resolution grids without optimization (e.g., spatial partitioning). The sharp cutoff may produce visual artifacts if the grid resolution is low [S1, unofficial].
*   **When to use:** Suitable for real-time applications where particle counts are moderate and grid resolution is manageable, or as a preprocessing step for implicit surface extraction [S1, unofficial].
*   **When not to use:** When smooth, infinite-support fields are required (use Gaussian instead) or when $N$ is very large without spatial acceleration structures.

## Background
The method is implemented in the context of GPU-based marching cubes and particle rendering, specifically within the Unity engine ecosystem using HLSL compute shaders [S1, S1, unofficial]. The specific kernel function $0.25 - m^2 + (m^2)^2$ with a cutoff at $m^2=0.5$ is designed to plot a curve that is $0.25$ at $r=0$ and $0$ at $r = \sqrt{0.5} \cdot \text{power}$, providing a "nice curve" for particle visualization [S1, unofficial]. The quantization variant is discussed in the context of particle-wise higher-order SPH field approximation to avoid rounding errors in recursive coefficient updates [S2].

## Sources
[S1] kristofe/UnityGPUMarchingCubes, https://github.com/kristofe/UnityGPUMarchingCubes, fetched 2026-10-26 (unofficial)
[S2] Particle-Wise Higher-Order SPH Field Approximation for DVR, https://arxiv.org/abs/2401.02896v1, fetched 2026-10-26
