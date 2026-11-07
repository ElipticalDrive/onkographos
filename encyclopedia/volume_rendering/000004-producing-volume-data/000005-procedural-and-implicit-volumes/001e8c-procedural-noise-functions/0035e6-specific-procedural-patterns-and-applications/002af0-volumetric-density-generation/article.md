# Volumetric Density Generation

Also called: Heterogeneous Volume Density Generation, Procedural Density Generation via Noise and Thresholding, getDensity

## Overview
Volumetric Density Generation creates a patchy, heterogeneous density field from procedural noise by applying a power function and a threshold to the raw noise output [S1, unofficial]. It solves the problem of generating sparse, organic volumetric structures (such as clouds or smoke) from continuous noise, distinguishing itself from siblings like [Sine-based procedural voxel generation](2477-sine-based-procedural-voxel-generation.md) and [Spherical Density Ramp with Noise](2610-spherical-density-ramp-with-noise.md) by relying on non-linear remapping and hard cutoffs rather than geometric ramps or trigonometric sampling [S1, unofficial].

## How it works
**Inputs and Outputs**
*   **Input:** A 3D position vector $\mathbf{p} = (x, y, z)$, a base density multiplier $d_{base}$, and FBM parameters (scale $s$, octaves $o$).
*   **Output:** A scalar density value $D \in [0, 1]$.

**Data Structures**
No persistent data structures are required; the method is stateless and computes density on-the-fly.

**Steps**
1.  **Compute Noise:** Evaluate a Fractal Brownian Motion (FBM) noise function at position $\mathbf{p}$ with scale $s$ and octaves $o$. Let the result be $n \in [-1, 1]$ [S1, unofficial].
    $$ n = \text{FBM}(\mathbf{p}, s, o) $$
2.  **Remap:** Linearly map the noise value from $[-1, 1]$ to $[0, 1]$ [S1, unofficial].
    $$ n' = n \cdot 0.5 + 0.5 $$
3.  **Power Function:** Apply a power function with exponent $e=2.5$ to suppress low values and create sparse regions [S1, unofficial].
    $$ d = (n')^e $$
4.  **Threshold:** If the density $d$ is below the threshold $t=0.15$, set it to zero [S1, unofficial].
    $$ d = \begin{cases} 0.0 & \text{if } d < 0.15 \\ d & \text{otherwise} \end{cases} $$
5.  **Scale and Clamp:** Multiply by the base density $d_{base}$ and clamp the result to $[0, 1]$ [S1, unofficial].
    $$ D = \min(1.0, \max(0.0, d \cdot d_{base})) $$

**Parameters**
*   **Scale ($s$):** 3.0 [S1, unofficial].
*   **Octaves ($o$):** 2 [S1, unofficial].
*   **Power Exponent ($e$):** 2.5 [S1, unofficial].
*   **Threshold ($t$):** 0.15 [S1, unofficial].
*   **Base Density ($d_{base}$):** 0.8 (default) [S1, unofficial].

**Edge Cases and Pitfalls**
*   **Precision:** The thresholding step creates discontinuities in the derivative of the density field, which may cause artifacts in gradient-based shading or marching cubes if not handled by smoothing [G].
*   **Boundaries:** The method assumes the input noise is bounded in $[-1, 1]$. If the FBM implementation returns values outside this range, the remap step will produce densities outside $[0, 1]$ before clamping [G].

**Pseudocode**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def volumetric_density_generation(pos, base_density=0.8, scale=3.0, octaves=2, exponent=2.5, threshold=0.15):
    # 1. Compute FBM noise
    noise = fbm(pos, scale, octaves)
    
    # 2. Remap from [-1, 1] to [0, 1]
    noise = noise * 0.5 + 0.5
    
    # 3. Apply power function
    density = pow(noise, exponent)
    
    # 4. Threshold
    if density < threshold:
        density = 0.0
        
    # 5. Scale and clamp
    density = density * base_density
    density = max(0.0, min(1.0, density))
    
    return density
```

## Subtopics

### [Volume Cloud Generation Pipeline](8286-volume-cloud-generation-pipeline.md)
Generates patchy volumetric density for clouds using multi-scale fractal noise, domain warping, and explicit boundary envelopes. Chosen when organic texture and vertical structure are required, diverging from the parent by adding warping and boundary handling rather than just power/threshold.

## Operating characteristics
*   **Time Complexity:** $O(o)$ per sample, where $o$ is the number of octaves in the FBM evaluation.
*   **Memory Complexity:** $O(1)$, as no volume data is stored.
*   **Strengths:** Extremely fast to compute; generates sparse, natural-looking structures without storing large voxel grids.
*   **Limitations:** The hard threshold creates non-smooth density gradients, which can lead to visual artifacts in ray marching or mesh extraction. It lacks the structural control of domain warping or geometric constraints.
*   **When to use:** Ideal for simple, sparse volumetric effects like distant clouds, smoke, or fog where performance is critical and high-fidelity structural control is not required.
*   **When not to use:** When smooth density transitions are needed, or when complex organic shapes require domain warping or boundary envelopes.

## Background
This method originates from practical implementations in volumetric path tracing, specifically documented in the GitHub repository `ainagutierrez01/Volumetric-Path-Tracing-` [S1, unofficial]. It represents a common heuristic in real-time and offline rendering for generating heterogeneous volumes from simple noise functions.

## Sources
[S1] ainagutierrez01/Volumetric-Path-Tracing-, https://github.com/ainagutierrez01/Volumetric-Path-Tracing-, fetched 2026-10-26 (unofficial)
