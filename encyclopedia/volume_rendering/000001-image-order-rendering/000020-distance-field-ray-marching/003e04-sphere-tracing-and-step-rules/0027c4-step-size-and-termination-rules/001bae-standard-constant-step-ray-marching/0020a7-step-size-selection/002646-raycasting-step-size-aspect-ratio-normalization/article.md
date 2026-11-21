# Raycasting Step Size & Aspect Ratio Normalization

Also called: Aspect Ratio Normalized Ray Marching, Fixed-Step Ray Marching Setup, Normalized Step Size Calculation

## Overview
This method establishes a uniform step size for volume rendering by normalizing the volume dataset's physical dimensions to a unit cube, ensuring consistent sampling density regardless of resolution or aspect ratio [S1, S2, unofficial]. It computes a base step size from the maximum volume dimension and an optional axis-wise scaling factor to map arbitrary voxel grids into normalized space [S1, S2, unofficial].

## How it works
**Inputs:**
*   Volume dimensions: $W$ (width), $H$ (height), $D$ (depth) in voxels [S1, S2, unofficial].
*   Optional aspect ratios: $\mathbf{a} = (a_x, a_y, a_z)$ defining relative scaling of axes [S2, unofficial].
*   Step scale multiplier: $s_{scale}$ [S2, unofficial].

**Outputs:**
*   Normalized step size: $\Delta$ [S1, S2, unofficial].
*   Scaling factor: $\mathbf{f}$ (used to map volume to unit cube) [S2, unofficial].

**Data Structures:**
*   `dimensions`: Tuple of three integers $(W, H, D)$.
*   `aspectRatios`: Tuple of three floats $(a_x, a_y, a_z)$.
*   `stepScale`: Single float.

**Numbered Steps:**
1.  **Calculate Max Dimension:** Determine the largest dimension of the volume.
    $$ M = \max(W, H, D) $$
    [S1, unofficial]
2.  **Compute Base Step Size:** Calculate the inverse of the maximum dimension.
    $$ \Delta_{base} = \frac{1}{M} $$
    [S1, unofficial]
3.  **Handle Aspect Ratios:** If aspect ratios are provided to handle non-cubic volumes explicitly:
    *   Clamp aspect ratios to a minimum of $\mathbf{0}$ [S2, unofficial].
    *   Compute the scaling factor $\mathbf{f}$ to map the volume into a unit cube coordinate system:
        $$ \mathbf{f} = \frac{\mathbf{1}}{\left(\mathbf{1} \cdot M\right) / (\mathbf{d} \cdot \mathbf{a})} $$
        Where $\mathbf{1} = (1,1,1)$, $\mathbf{d} = (W,H,D)$, and $\mathbf{a} = (a_x, a_y, a_z)$ [S2, unofficial]. Vector operations are component-wise [G]. The source computes per-axis step size as $1.0 / (d_i \cdot (M/d_i))$, which algebraically simplifies to $1/M$, confirming the step size remains uniform while the scaling factor adjusts the spatial mapping [S2, unofficial].
4.  **Apply Step Scale:** Multiply the base step size by the user-defined scale factor, clamped to a minimum of $0.1$ [S2, unofficial].
    $$ \Delta = \max(\Delta_{base} \cdot s_{scale}, 0.1) $$
    [S2, unofficial]
5.  **Pass to Shader:** Upload $\Delta$ (and $\mathbf{f}$ if used) as uniform variables to the ray marching shader [S1, S2, unofficial].

**Parameters:**
*   `stepScale`: Multiplier for the base step size during ray marching. Clamped to a minimum of $0.1$ [S2, unofficial].
*   `aspectRatios`: Vector defining the relative scaling of volume axes. Clamped to a minimum of $\mathbf{0}$ [S2, unofficial].
*   `_fovy`: Vertical field of view in radians (used in the Projection Factor variant) [S3, unofficial].
*   `_pixelTolerance`: Desired pixel tolerance for step sizing (used in the Projection Factor variant) [S3, unofficial].
*   `screenDivisor`: Resolution divisor, default $1.0$ or $8.0$ for min-depth pass (used in the Projection Factor variant) [S3, unofficial].

**Edge Cases and Pitfalls:**
*   **Zero Dimensions:** If any dimension is $0$, the calculation would divide by zero; a guard clause returns a default step size of $1.0$ [G].
*   **Zero Aspect Ratios:** Clamping to $0$ is specified in the source [S2, unofficial], but zero values in the denominator of the scale factor calculation will produce infinite scaling unless the renderer's coordinate mapping logic handles it [G].
*   **Precision:** Using `float` for step size is standard in shaders; host-side calculations should match shader precision [G].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def compute_step_size_and_scale(width, height, depth, aspect_ratios=None, step_scale=1.0):
    """
    Computes the normalized step size and scaling factor for volume ray marching.
    
    Args:
        width (int): Volume width in voxels.
        height (int): Volume height in voxels.
        depth (int): Volume depth in voxels.
        aspect_ratios (tuple): Optional (ax, ay, az). Defaults to (1, 1, 1).
        step_scale (float): Multiplier for the base step size.
        
    Returns:
        tuple: (step_size, scale_factor)
    """
    # 1. Calculate max dimension
    max_dim = max(width, height, depth)
    
    if max_dim == 0:
        return 1.0, (1.0, 1.0, 1.0) # Guard against division by zero [G]

    # 2. Base step size
    base_step = 1.0 / max_dim
    
    # 3. Aspect ratio handling
    if aspect_ratios is None:
        aspect_ratios = (1.0, 1.0, 1.0)
        
    # Clamp aspect ratios to minimum 0 [S2, unofficial]
    ax, ay, az = max(aspect_ratios[0], 0.0), max(aspect_ratios[1], 0.0), max(aspect_ratios[2], 0.0)
    
    dimensions = (float(width), float(height), float(depth))
    
    # Compute scale factor: vec3(1) / ((vec3(1) * maxSize) / (dimensions * aspectRatios)) [S2, unofficial]
    # Component-wise division [G]
    scale_x = 1.0 / ((1.0 * max_dim) / (dimensions[0] * ax))
    scale_y = 1.0 / ((1.0 * max_dim) / (dimensions[1] * ay))
    scale_z = 1.0 / ((1.0 * max_dim) / (dimensions[2] * az))
    
    scale_factor = (scale_x, scale_y, scale_z)
    
    # 4. Apply step scale and clamp to min 0.1 [S2, unofficial]
    final_step = max(base_step * step_scale, 0.1)
    
    return final_step, scale_factor
```

## Subtopics

### [Uniform Ray Marching](4810-uniform-ray-marching.md)
Uniform Ray Marching generates 3D sample positions along rays within an Axis-Aligned Bounding Box (AABB) using a fixed spacing between samples. It serves as a fallback strategy for environments lacking adaptive libraries or as a debug mode that treats an atlas texture as the full volume without brick traversal. It replaces coarse sampling to avoid aliasing artifacts, such as concentric "sampling rings," caused by steep gradients near inner radius cuts.

### Projection Factor Calculation
Calculates a scaling factor for ray marching step sizes based on camera FOV and pixel tolerance to ensure consistent visual quality across resolutions [S3, unofficial]. The formula is $F = \frac{1}{2 \tan(\frac{\theta}{2})} \cdot \frac{W_y}{\text{pixelTolerance}}$ where $\theta$ is vertical FOV, $W_y$ is screen height, and pixelTolerance is in pixels [S3, unofficial]. Parameters include `_fovy` (vertical FOV in radians), `_pixelTolerance` (desired pixel tolerance), and `screenDivisor` (resolution divisor, default $1.0$ or $8.0$ for min-depth pass) [S3, unofficial].

## Operating characteristics
*   **Time Complexity:** $O(1)$ for step size calculation [G]. Ray marching traversal is $O(N)$ where $N$ is the step count [G].
*   **Memory Complexity:** $O(1)$ for the step size and scale factor variables [G].
*   **Performance:** Provides consistent rendering speed as step count is predictable [G].
*   **Strengths:** Simple to implement; ensures consistent sampling density across different volume resolutions [S1, S2, unofficial].
*   **Limitations:** Does not adapt to local detail [G].
*   **When to Use:** When consistent sampling density is critical or when hardware/software constraints prevent adaptive stepping [S2, unofficial].

## Background
This method is derived from practical implementations in open-source volume rendering projects, specifically `qt-viewer-pro` and `VolumeRendering` [S1, S2, unofficial]. It addresses the common problem of visual artifacts that arise when ray marching through volumes with varying aspect ratios or resolutions using naive fixed steps [S2, unofficial]. A related variant calculates step scaling based on camera FOV and pixel tolerance to maintain visual quality across resolutions [S3, unofficial].

## Sources
[S1] wpop/qt-viewer-pro, https://github.com/wpop/qt-viewer-pro, fetched 2026-10-27 (unofficial)
[S2] jose-villegas/VolumeRendering, https://github.com/jose-villegas/VolumeRendering, fetched 2026-10-27 (unofficial)
[S3] RvanderLaan/SVDAG-Compression, https://github.com/RvanderLaan/SVDAG-Compression, fetched 2026-10-26 (unofficial)
