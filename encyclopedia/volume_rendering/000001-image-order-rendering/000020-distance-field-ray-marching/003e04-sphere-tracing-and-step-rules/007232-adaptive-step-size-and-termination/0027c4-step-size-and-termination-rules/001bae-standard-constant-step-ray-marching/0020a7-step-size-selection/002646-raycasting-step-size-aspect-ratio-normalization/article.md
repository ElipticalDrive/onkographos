# Raycasting Step Size & Aspect Ratio Normalization

Also called: Aspect Ratio Normalized Ray Marching, Fixed-Step Ray Marching Setup, Fixed-Step Ray Marching with Steps Per Voxel, Fixed-Steps, Normalized Step Size Calculation, Step Size Calculation, Step-based voxel raymarching

## Overview
This method establishes a uniform step size for volume rendering by normalizing the volume dataset's physical dimensions to a unit cube, ensuring consistent sampling density regardless of the volume's aspect ratio or resolution [S1, S3, unofficial]. It sets itself apart from adaptive methods like [Adaptive Step Size Calculation](4671-adaptive-step-size-calculation.md) by using constant intervals rather than density-based acceleration, and from [Dartmouth Distance Sampling](7211-dartmouth-distance-sampling.md) by relying on fixed increments rather than analytical intersection math [S4, unofficial].

## How it works
**Inputs:**
- Volume dimensions: $W$ (width), $H$ (height), $D$ (depth) in voxels [S1, S5, unofficial].
- Optional aspect ratio scaling factors: $\mathbf{a} = (a_x, a_y, a_z)$ [S3, unofficial].
- Step scale multiplier: $s_{scale}$ [S3, S5, unofficial].
- Maximum ray distance: $d_{max}$ [S4, unofficial].
- Camera ray origin: $\mathbf{p}_0$ [S4, unofficial].
- Camera ray direction: $\mathbf{d}$ (normalized) [S4, unofficial].

**Outputs:**
- Step size vector: $\mathbf{s}$ [S3, S5, unofficial].
- Marching results: Hit coordinates and previous coordinates [S4, unofficial].

**Data Structures:**
- `stepSize`: A 3-component vector (float32) representing the distance to advance along each axis [S3, S5, unofficial].
- `curr_pos`: A 3-component vector (float32) tracking the current ray position [S4, unofficial].

**Steps:**
1. **Calculate Max Dimension:** Determine the largest dimension of the volume dataset.
   $$ M = \max(W, H, D) $$
   [S1, S5, unofficial]

2. **Compute Base Step Size:** Calculate the step size for each axis. The sources provide two variations for this calculation.
   
   *Variation A (Simple Normalization):*
   $$ \mathbf{s}_{base} = \left( \frac{1}{M}, \frac{1}{M}, \frac{1}{M} \right) $$
   [S1, unofficial]
   
   *Variation B (Aspect-Ratio Aware):*
   If aspect ratios $\mathbf{a}$ are provided, compute a scaling factor to map arbitrary dimensions into a unit cube.
   $$ \mathbf{s}_{base} = \left( \frac{1}{W \cdot (M/W)}, \frac{1}{H \cdot (M/H)}, \frac{1}{D \cdot (M/D)} \right) $$
   [S3, S5, unofficial]
   Note: In Variation B, the term $(M/W)$ effectively normalizes the axis length relative to the max dimension. If aspect ratios are used to adjust the scale factor, the formula becomes:
   $$ \text{scaleFactor} = \frac{1}{(1 \cdot M) / (\mathbf{d} \cdot \mathbf{a})} $$
   where $\mathbf{d}$ is the dimension vector $(W, H, D)$ [S3, unofficial].

3. **Apply Step Scale:** Multiply the base step size by the user-defined scale factor $s_{scale}$ (default 1.0) [S5, unofficial].
   $$ \mathbf{s} = \mathbf{s}_{base} \cdot s_{scale} $$
   [S5, unofficial]
   The step scale is typically clamped to a minimum of 0.1f to prevent excessively small steps [S3, unofficial].

4. **Initialize Ray Marching:**
   - Set current position $\mathbf{p}_{curr} = \mathbf{p}_0$ [S4, unofficial].
   - Normalize the look vector $\mathbf{d}$. If $\|\mathbf{d}\| < 10^{-5}$, terminate immediately [S4, unofficial].
   - Calculate maximum iterations: $N_{max} = \lceil d_{max} / \|\mathbf{s}\| \rceil$ [S4, unofficial]. Note: Some implementations use $M \cdot (1/s_{scale}) \cdot 2$ as a heuristic for iterations [S5, unofficial].

5. **Marching Loop:**
   For $i$ from 0 to $N_{max}$:
   - Check voxel at $\mathbf{p}_{curr}$. If a solid voxel is detected, return $(\mathbf{p}_{curr}, \mathbf{p}_{prev})$ and stop [S4, unofficial].
   - Store $\mathbf{p}_{prev} = \mathbf{p}_{curr}$ [S4, unofficial].
   - Advance position: $\mathbf{p}_{curr} = \mathbf{p}_{curr} + \mathbf{s} \cdot \|\mathbf{d}\|$ (or simply $\mathbf{p}_{curr} + \mathbf{s}$ if $\mathbf{d}$ is already normalized and $\mathbf{s}$ is direction-agnostic) [S4, unofficial].
   - If $\mathbf{p}_{curr}$ exceeds bounds or distance limit, terminate [S4, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import numpy as np

def calculate_step_size(width: int, height: int, depth: int, step_scale: float = 1.0) -> np.ndarray:
    """
    Calculate normalized step size vector for ray marching.
    Based on [S1, S3, S5, unofficial].
    """
    # 1. Calculate max dimension
    max_dim = max(width, height, depth)
    
    if max_dim == 0:
        return np.array([0.0, 0.0, 0.0])
    
    # 2. Compute base step size (Variation B from [S3, S5, unofficial])
    # Note: The formula 1.0 / (dim * (max_dim / dim)) simplifies to 1.0 / max_dim
    # However, we retain the structure if aspect ratios are intended to be applied later.
    # For pure normalization without aspect ratios, this is uniform.
    step_x = 1.0 / (width * (max_dim / width))
    step_y = 1.0 / (height * (max_dim / height))
    step_z = 1.0 / (depth * (max_dim / depth))
    
    step_size = np.array([step_x, step_y, step_z], dtype=np.float32)
    
    # 3. Apply step scale
    step_size *= step_scale
    
    # Clamp minimum step scale if necessary (implementation detail from [S3, unofficial])
    # step_size = np.maximum(step_size, 0.1) # Usually applied to scale factor, not step directly
    
    return step_size

def perform_raycast(
    start_pos: np.ndarray,
    look_vector: np.ndarray,
    step_size: np.ndarray,
    max_dist: float = 8.0,
    voxel_checker=None
) -> tuple:
    """
    Perform fixed-step ray marching.
    Based on [S4, unofficial].
    """
    # Normalize look vector
    norm = np.linalg.norm(look_vector)
    if norm < 1e-5:
        return None
    
    direction = look_vector / norm
    
    # Initialize
    curr_pos = start_pos.copy()
    prev_pos = start_pos.copy()
    
    # Calculate max iterations based on max_dist and step magnitude
    # Using the magnitude of the step size vector as the increment distance
    step_mag = np.linalg.norm(step_size)
    if step_mag == 0:
        return None
        
    max_iterations = int(max_dist / step_mag) + 1
    
    for _ in range(max_iterations):
        # Check voxel at current position
        # voxel_checker should return True if solid voxel is detected
        if voxel_checker is not None and voxel_checker(curr_pos):
            return (curr_pos, prev_pos)
        
        prev_pos = curr_pos.copy()
        
        # Advance along the direction by the step size
        # Note: step_size is a vector, direction is a vector.
        # In many implementations, step_size is isotropic or applied along the ray direction.
        # Based on [S4, unofficial], it advances by step_size along the normalized look vector.
        # If step_size is a scalar magnitude, it would be: curr_pos += direction * step_size
        # If step_size is a vector representing the max axis step, it needs projection.
        # The sources [S4, unofficial] imply a scalar step_size or a vector applied directly.
        # Assuming step_size is the magnitude to advance per step along the ray:
        curr_pos = curr_pos + direction * step_mag
        
    return None
```

**Parameters:**
- `fixedStepSize`: Boolean uniform enabling fixed step size ray marching. Set to `true` [S2, unofficial].
- `stepsPerVoxel`: Integer uniform controlling ray step granularity relative to voxel size. Typical value is 2, meaning each ray step covers half a voxel [S2, unofficial].
- `stepScale`: Multiplier for the base step size. Clamped to a minimum of 0.1f [S3, unofficial].
- `aspectRatios`: `vec3` defining relative scaling of volume axes. Clamped to a minimum of `vec3(0)` [S3, unofficial].
- `max_dist`: Maximum distance the ray travels. Typically 8.0 blocks [S4, unofficial].
- `step_size`: Increment distance. Typically 0.1 blocks [S4, unofficial].

**Edge Cases and Pitfalls:**
- **Zero Dimensions:** If $W, H,$ or $D$ is 0, division by zero occurs in step size calculation. Guard against this [S1, unofficial].
- **Degenerate Ray:** If the look vector norm is near zero, the ray direction is undefined. Return immediately [S4, unofficial].
- **Precision:** Using `float32` for positions and step sizes is standard, but accumulation of error over many steps may require periodic re-normalization or double precision for long rays [G].
- **Aspect Ratio Distortion:** Without aspect ratio normalization, non-cubic volumes will have stretched sampling, leading to aliasing or gaps [S3, unofficial].

## Subtopics

### [Uniform Ray Marching](4810-uniform-ray-marching.md)
Uniform Ray Marching is a sampling strategy that generates 3D sample positions along rays within an Axis-Aligned Bounding Box (AABB) using a fixed spacing between consecutive samples. It serves as a debug rendering mode for direct atlas sampling without brick traversal, a PyTorch-native fallback for environments lacking custom CUDA extensions, and a dense uniform sampler for Line-of-Sight (LOS) integral visualization. It sets itself apart from adaptive methods like [NeRF Ray Marching Optimizations](9999-nerf-ray-marching-optimizations.md) by using constant intervals rather than density-based acceleration, and from [Oversampled 3D Ray March](5708-oversampled-3d-ray-march.md) by not enforcing strict slab ownership at LOD boundaries.

### Projection Factor Calculation
Calculates a scaling factor for ray marching step sizes based on camera FOV and pixel tolerance to ensure consistent visual quality across resolutions [S6, unofficial].
**Formula:**
$$ F = \frac{1}{2 \tan(\frac{\theta}{2})} \cdot \frac{W_y}{\text{pixelTolerance}} $$
where $\theta$ is vertical FOV, $W_y$ is screen height, and pixelTolerance is in pixels [S6, unofficial].
**Parameters:**
- `_fovy`: Vertical field of view in radians [S6, unofficial].
- `_pixelTolerance`: Desired pixel tolerance for step sizing [S6, unofficial].
- `screenDivisor`: Resolution divisor (default 1.0f, or 8.0f for min-depth pass) [S6, unofficial].

## Operating characteristics
- **Time Complexity:** $O(N)$ where $N$ is the number of steps along the ray. $N$ is determined by $d_{max} / \text{step\_size}$.
- **Memory Complexity:** $O(1)$ auxiliary memory per ray, excluding the volume texture storage.
- **Performance:** Fixed-step marching is computationally predictable and easy to parallelize on GPUs [S2, unofficial]. However, it is less efficient than adaptive methods in sparse volumes because it samples empty space uniformly [S1, unofficial].
- **Strengths:** Simple to implement, deterministic, no branching overhead for step size adjustment [S4, unofficial].
- **Limitations:** Inefficient for large empty regions; requires careful tuning of `stepScale` to balance quality and speed [S3, unofficial].
- **When to Use:** Debugging, uniform density volumes, or when hardware lacks advanced branching capabilities [S2, S4, unofficial].
- **When Not to Use:** Sparse volumes where adaptive methods like [Adaptive Step Size Calculation](4671-adaptive-step-size-calculation.md) would skip empty space efficiently.

## Background
Originating from early voxel-based rendering techniques, this method was popularized in game engines for block-based worlds (e.g., Minecraft-like environments) due to its simplicity and robustness [S4, unofficial]. It is widely used in medical imaging viewers (e.g., [qt-viewer-pro](https://github.com/wpop/qt-viewer-pro)) for consistent visualization across different dataset resolutions [S1, unofficial]. The aspect ratio normalization component addresses the issue of distorted sampling in non-cubic volumes, a common problem in medical and scientific datasets [S3, unofficial].

## Sources
[S1] wpop/qt-viewer-pro, https://github.com/wpop/qt-viewer-pro, fetched 2026-10-27 (unofficial)
[S2] aryaman-gupta/LiV-renderer, https://github.com/aryaman-gupta/LiV-renderer, fetched 2026-10-27 (unofficial)
[S3] jose-villegas/VolumeRendering, https://github.com/jose-villegas/VolumeRendering, fetched 2026-10-27 (unofficial)
[S4] herbal1st/pyvorengi-sdk-demo, https://github.com/herbal1st/pyvorengi-sdk-demo, fetched 2026-10-27 (unofficial)
[S5] 3D Texture Only interpolates on Y-Axis, https://gamedev.stackexchange.com/questions/166205, fetched 2026-10-29 (unofficial)
[S6] RvanderLaan/SVDAG-Compression, https://github.com/RvanderLaan/SVDAG-Compression, fetched 2026-10-26 (unofficial)
