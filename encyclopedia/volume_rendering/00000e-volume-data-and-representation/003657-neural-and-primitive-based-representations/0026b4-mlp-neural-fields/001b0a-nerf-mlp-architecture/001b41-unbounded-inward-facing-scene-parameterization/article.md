# Unbounded inward-facing scene parameterization

Also called: Mip-NeRF 360 parameterization, Scene Contraction, Mip-NeRF 360 Scene Contraction, Contracted space parameterization, Mip-NeRF 360 contraction

## Overview
This method parameterizes unbounded 3D scenes by mapping coordinates from $[-\infty, \infty]^d$ to a bounded domain (typically $[-2, 2]^d$), enabling neural radiance fields and voxel grids to handle large-scale or infinite environments [S1][S2, unofficial][S3]. It solves the problem of sparse sampling in distant regions by compressing far-field coordinates into a finite volume, allowing the network to allocate model capacity proportional to disparity (inverse distance) [S1][S2, unofficial]. It differs from standard bounding-box parameterizations by preserving the radial direction of input vectors while non-linearly compressing their magnitude, ensuring that points at infinity map to the surface of a sphere or cuboid [S2, unofficial].

## How it works
**Inputs and Outputs:**
- Input: 3D coordinate vector $\mathbf{x} \in \mathbb{R}^3$ (or $\mathbb{R}^d$).
- Output: Contracted coordinate vector $\mathbf{x}' \in \mathbb{R}^3$ within a bounded volume.

**Data Structures:**
- The method operates on standard floating-point vectors. No specific complex data structures are required for the transformation itself, though it is typically applied to coordinates before querying occupancy grids or neural fields [S2, unofficial][S4, unofficial].

**Parameters:**
- $p$: The norm used for contraction. Common values are $p=2$ (Euclidean) or $p=\infty$ (Chebyshev/L-infinity) [S2, unofficial].
- $b$: A hyperparameter controlling the proportion of the grid allocated to the background. $b > 0$ [S2, unofficial].
- $d$: The dimensionality of the space. Typically $d=3$ [S1].

**Formulas:**

1. **Standard Spherical Contraction (Mip-NeRF 360):**
   Maps $\mathbb{R}^3$ to a sphere of radius 2 [S1][S3]:
   $$
   \operatorname{contract}(\mathbf{x}) = \begin{cases} 
   \mathbf{x} & \|\mathbf{x}\| \leq 1 \\ 
   \left(2 - \frac{1}{\|\mathbf{x}\|}\right)\left(\frac{\mathbf{x}}{\|\mathbf{x}\|}\right) & \|\mathbf{x}\| > 1 
   \end{cases}
   $$
   where $\|\mathbf{x}\|$ is the Euclidean norm $\|\mathbf{x}\|_2$ [S1][S3].

2. **Generalized Cuboid/Spherical Contraction (DVGOv2):**
   Allows for a cuboid output using $L_p$ norm and background parameter $b$ [S2, unofficial]:
   $$
   \mathbf{x}' = \begin{cases} 
   \mathbf{x} & \|\mathbf{x}\|_p \leq 1 \\ 
   \left(1 + b - \frac{b}{\|\mathbf{x}\|_p}\right) \left(\frac{\mathbf{x}}{\|\mathbf{x}\|_p}\right) & \|\mathbf{x}\|_p > 1 
   \end{cases}
   $$
   where $\|\mathbf{x}\|_p$ is the $L_p$ norm [S2, unofficial].

3. **Piecewise-Projective Contraction (MERF):**
   Preserves straight lines for efficient ray-AABB intersection [S5]:
   $$
   \operatorname{contract}_{\pi}(\mathbf{x})_j = \begin{cases} 
   x_j & \text{if } \|\mathbf{x}\|_{\infty} \leq 1 \\ 
   \frac{x_j}{\|\mathbf{x}\|_{\infty}} & \text{if } x_j \neq \|\mathbf{x}\|_{\infty} > 1 \\ 
   \left(2 - \frac{1}{|x_j|}\right) & \text{if } x_j = \|\mathbf{x}\|_{\infty} > 1 
   \end{cases}
   $$
   where $\|\mathbf{x}\|_{\infty} = \max_i |x_i|$ and $j$ is the coordinate index [S5].

**Steps for Implementation:**

1. **Preprocessing (Optional but recommended for DVGOv2):**
   - Shift the world coordinates so that the camera centroid is at the origin [S2, unofficial].
   - Scale the coordinates so that the cameras lie near the unit sphere planes [S2, unofficial].
   - (Optional) Rotate the coordinates to align the first two PCA directions of camera positions with the grid's XY axis [S2, unofficial].

2. **Coordinate Transformation:**
   - Compute the norm $\|\mathbf{x}\|_p$ of the input coordinate $\mathbf{x}$.
   - If $\|\mathbf{x}\|_p \leq 1$, return $\mathbf{x}' = \mathbf{x}$.
   - If $\|\mathbf{x}\|_p > 1$, compute the contracted coordinate using the selected formula (Standard, Generalized, or Piecewise-Projective).

3. **Grid Querying:**
   - Use the transformed coordinate $\mathbf{x}'$ to query the occupancy grid or neural field [S4, unofficial].
   - If using a voxel grid, ensure the grid is allocated to cover the bounded domain (e.g., length $2+2b$ for the generalized contraction) [S2, unofficial].

**Pseudocode for Standard Spherical Contraction:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import math

def contract_spherical(x):
    """
    Contracts unbounded 3D space to a sphere of radius 2.
    Args:
        x: list or tuple of 3 floats (x, y, z)
    Returns:
        list of 3 floats representing the contracted coordinates
    """
    # Calculate Euclidean norm
    norm = math.sqrt(x[0]**2 + x[1]**2 + x[2]**2)
    
    # If inside unit sphere, return unchanged
    if norm <= 1.0:
        return x
    
    # If outside, apply contraction
    # Factor = (2 - 1/norm)
    factor = 2.0 - (1.0 / norm)
    
    # Scale the vector by factor / norm (which is equivalent to factor * (x / norm))
    scale = factor / norm
    
    return [x[0] * scale, x[1] * scale, x[2] * scale]
```

**Pseudocode for Generalized Contraction:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import math

def contract_generalized(x, p=2, b=0.0):
    """
    Contracts unbounded 3D space using Lp norm and background parameter b.
    Args:
        x: list or tuple of 3 floats
        p: norm type (2 for Euclidean, math.inf for L-infinity)
        b: background proportion hyperparameter (b > 0)
    Returns:
        list of 3 floats
    """
    # Calculate Lp norm
    if p == math.inf:
        norm = max(abs(v) for v in x)
    else:
        norm = sum(abs(v)**p for v in x)**(1.0/p)
    
    # If inside unit sphere, return unchanged
    if norm <= 1.0:
        return x
    
    # If outside, apply contraction
    # Factor = (1 + b - b/norm)
    factor = 1.0 + b - (b / norm)
    
    # Scale the vector
    scale = factor / norm
    
    return [x[0] * scale, x[1] * scale, x[2] * scale]
```

**Edge Cases and Pitfalls:**
- **Precision:** When $\|\mathbf{x}\|$ is very close to 1, ensure the transition is smooth. The formulas are continuous at $\|\mathbf{x}\|=1$ [S1][S2, unofficial].
- **Norm Selection:** Using $p=2$ with a cuboid voxel grid wastes approximately 50% of grid points because a sphere occupies only a fraction of a cube [S2, unofficial]. Using $p=\infty$ creates a cuboid contracted space, which is more efficient for voxel grids [S2, unofficial].
- **Ray Intersections:** Standard spherical contraction maps straight lines to curves, making ray-AABB intersection tests for empty space skipping challenging [S5]. Use Piecewise-Projective Contraction if efficient ray-AABB intersection is required [S5].
- **Unbounded Support:** Standard Occupancy Grids (e.g., from Instant-NGP) only support bounded scenes; scene contraction is required to extend them to unbounded scenes [S4, unofficial].

## Subtopics

### [Scene Contraction](9762-scene-contraction.md)
Scene contraction is a coordinate transformation technique that maps unbounded 3D space into a bounded volume, enabling neural radiance fields to handle large-scale or infinite environments. It solves the problem of sparse sampling in distant regions by compressing far-field coordinates into a finite ball, allowing the network to learn a compact representation of the entire scene. It differs from standard bounding-box parameterizations by preserving the radial direction of input vectors while non-linearly compressing their magnitude, ensuring that points at infinity map to the surface of the sphere. Choose this method when you need to represent unbounded scenes with a neural field or voxel grid that requires bounded inputs. See [Scene Contraction](9762-scene-contraction.md).

### Forward-facing scene parameterization
A method to handle unbounded forward-facing scenes by warping the frustum to a bounded volume using a dense voxel grid allocation similar to Multiplane Images (MPIs) [S2, unofficial]. It places $D$ RGB-density images at fixed depths, each with $X \times Z$ resolution, and traces rays through these layers [S2, unofficial]. Choose this method for forward-facing scenes (e.g., landscape photography) where the camera motion is constrained to a plane or frustum, rather than full 360-degree inward-facing capture.

### Piecewise-Projective Contraction
A contraction function that maps unbounded space into a finite domain while preserving straight lines (piecewise-linear), allowing for efficient ray-AABB intersection tests for empty space skipping [S5]. It consists of seven regions (unit cube + 6 outer regions) [S5]. Unlike spherical contraction which maps to a radius-2 ball, this contraction preserves the unit cube and maps other regions via projective transformations [S5]. It performs on par with spherical contraction but enables efficient empty space skipping [S5]. Choose this method when real-time rendering with empty space skipping is critical and you are using a voxel-based representation.

## Operating characteristics
- **Time Complexity:** The contraction function itself is $O(1)$ per coordinate, involving basic arithmetic operations (norm calculation, division, multiplication) [S1][S2, unofficial][S5].
- **Memory Complexity:** The method itself does not increase memory usage; it enables the use of bounded voxel grids for unbounded scenes, which can significantly reduce memory compared to bounding the entire unbounded space with a large box [S2, unofficial][S4, unofficial].
- **Measured Performance:** DVGOv2 using this parameterization achieved comparable quality to Plenoxels and TensoRF with fewer voxels ($384^2 \times 256$ vs $1408 \times 1156 \times 128$ or $640^3$) on the LLFF dataset [S2, unofficial]. On Tanks&Temples, it achieved competitive results with smaller model sizes [S2, unofficial].
- **Scaling:** The method scales well to large scenes because it allocates higher representational capacity to points near the cameras (high disparity) and lower capacity to distant points [S1][S3].
- **Strengths:** Enables representation of unbounded scenes with bounded data structures; efficient for voxel grids when $p=\infty$ is used; preserves radial direction [S1][S2, unofficial].
- **Limitations:** Standard spherical contraction distorts straight lines, complicating ray-AABB intersections [S5]; requires careful selection of norm and hyperparameters for optimal grid utilization [S2, unofficial].
- **When to use:** When rendering unbounded inward-facing scenes (360-degree photos) or forward-facing scenes with a voxel-based or grid-based neural representation.
- **When not to use:** When the scene is strictly bounded and small enough to fit in a simple bounding box without significant waste; when real-time ray-AABB intersection is required and spherical contraction is used (use Piecewise-Projective instead).

## Background
The scene contraction idea originates from Mip-NeRF 360 [S4, unofficial][S1]. Mip-NeRF 360 introduced a spatial contraction function to parameterize unbounded scenes with bounded coordinates, mapping $[-\infty, \infty]^d$ to $[-2, 2]^d$ [S1][S4, unofficial]. This approach was adapted by DVGOv2 for voxel grid allocation in unbounded 360-degree scenes [S2, unofficial]. The concept has since been integrated into acceleration toolboxes like NerfAcc to support unbounded scenes with Instant-NGP's occupancy grids [S4, unofficial]. Variants like Piecewise-Projective Contraction were proposed by MERF to address the limitation of spherical contraction regarding ray-AABB intersections [S5].

## Sources
[S1] Zip-NeRF: Anti-Aliased Grid-Based Neural Radiance Fields, https://doi.org/10.1109/iccv51070.2023.01804, fetched 2026-10-26
[S2] Improved Direct Voxel Grid Optimization for Radiance Fields Reconstruction, https://doi.org/10.48550/arxiv.2206.05085, fetched 2026-10-26 (unofficial)
[S3] NeRF-Casting: Improved View-Dependent Appearance with Consistent Reflections, https://arxiv.org/abs/2405.14871v1, fetched 2026-10-26
[S4] NerfAcc: A General NeRF Acceleration Toolbox, https://doi.org/10.48550/arxiv.2210.04847, fetched 2026-10-26 (unofficial)
[S5] MERF: Memory-Efficient Radiance Fields for Real-time View Synthesis in Unbounded Scenes, https://doi.org/10.1145/3592426, fetched 2026-10-26
