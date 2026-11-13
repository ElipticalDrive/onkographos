# Boundary-structure-aware transfer function

Also called: BSATF

## Overview
Boundary-structure-aware transfer functions (BSATF) is a volume classification method that combines boundary-based and structure-based advantages by using the standard deviation of ambient occlusion to quantify variation of boundary and structure information across voxels [S1]. It solves the problem of separating materials with similar average intensities but different intensity variations or global structures, which standard ambient occlusion and gradient-based methods fail to distinguish [S1]. Unlike [Boundary-Enhanced Opacity](795-boundary-enhanced-opacity.md) which focuses on local boundary enhancement, or [Data Feature-Based Classification](5934-data-feature-based-classification.md) which relies on local derivatives, BSATF leverages global structural context via the standard deviation of occlusion to separate objects with identical intensities and sizes but different locations or internal variations [S1].

## How it works
**Inputs and Outputs**
*   **Input:** A scalar volume field $f(x)$, where $x$ represents voxel coordinates.
*   **Output:** A 2D classification space (histogram) mapping intensity (or second derivative) and the standard deviation of ambient occlusion of the second derivative field to color and opacity [S1].

**Data Structures**
*   **Scalar Field $f(x)$:** The input volume data.
*   **Divergence Field $f''(x)$:** A derived field where $f''(x) = \nabla \cdot \nabla f(x)$ (the Laplacian). This field suppresses continuously changed intensity into a low constant value and highlights boundaries [S1].
*   **Ambient Occlusion Field $O(p, \vec{d})$:** Computed per voxel $p$ using input field $\vec{d}$ (typically $f''(x)$ or $f(x)$) [S1].
*   **Standard Deviation Field $O_{std}(p, \vec{d})$:** The standard deviation of directional occlusions for each voxel [S1].
*   **2D Histogram:** Axes are defined as:
    *   Horizontal: Intensity $f(x)$ or Divergence $f''(x)$ [S1].
    *   Vertical: Standard deviation of ambient occlusion $O_{std}(f''(x))$ [S1].
*   **Transfer Function Mappings:** Three 1D mappings for color and opacity based on $f(x)$, $f''(x)$, and $O_{std}(f''(x))$ [S1].

**Parameters**
*   $T$: Number of samples along the direction $\hat{\omega}$ for computing directional occlusion [S1].
*   $H$: Number of different directions used to compute the standard deviation of ambient occlusion [S1].
*   $M(p, \vec{d})$: Visibility mapping function, application-specific (e.g., binary for isosurface, Gaussian weighted, or linear ramp) [S1].
*   $\vec{d}$: The input field used for occlusion calculation. Can be $f(x)$ or $f''(x)$ [S1].

**Steps**
1.  **Compute Divergence:** Calculate the divergence of the gradient of the input field $f(x)$ to obtain $f''(x)$. This step suppresses continuously changed intensity into a low constant value and highlights boundaries [S1].
    $$ f''(x) = \nabla \cdot \nabla f(x) $$
2.  **Select Input Field for Occlusion:** Choose the input field $\vec{d}$ for ambient occlusion calculation. Typically, $\vec{d} = f''(x)$ is used to leverage global structure information on boundaries, though $f(x)$ may be used for simple cases with few objects [S1].
3.  **Compute Directional Occlusion:** For each voxel $p$ and direction $\hat{\omega}$, compute the directional occlusion $A(p, \hat{\omega}, \vec{d})$ by summing visibility values along the ray [S1]:
    $$ A(p, \hat{\omega}, \vec{d}) = \sum_{t=0}^{T} M(p + t \hat{\omega}, \vec{d}) $$
    where $T$ is the number of samples along $\hat{\omega}$ and $M$ is the visibility mapping function [S1].
4.  **Compute Ambient Occlusion:** Approximate the ambient occlusion $O(p, \vec{d})$ by averaging directional occlusions over $N$ neighbors (or directions) [S1]:
    $$ O(p, \vec{d}) \approx \frac{2\pi}{N} \sum_{\phi=0}^{2\pi} \sum_{\theta=0}^{\pi} A(p, \hat{\omega}(\theta, \phi), \vec{d}) $$
    Note: The standard deviation calculation uses the individual directional occlusions rather than this average sum [S1].
5.  **Compute Standard Deviation of Ambient Occlusion:** Calculate $O_{std}(p, \vec{d})$ for each voxel $p$ using $H$ different directions $\hat{\omega}_i$ [S1]:
    $$ O_{std}(p, \vec{d}) = \sqrt{ \frac{1}{H} \sum_{i=1}^{H} (A(p, \hat{\omega}_i, \vec{d}))^2 - \left( \frac{1}{H} \sum_{i=1}^{H} A(p, \hat{\omega}_i, \vec{d}) \right)^2 } $$
    This quantifies the variation of boundary and structure information, allowing separation of objects with same intensity but different surroundings or locations [S1].
6.  **Construct 2D Classification Space:** Build a 2D histogram with the horizontal axis as $f(x)$ or $f''(x)$ and the vertical axis as $O_{std}(f''(x))$ [S1].
7.  **Tag Regions and Assign Color/Opacity:** Tag regions in the histogram. Assign color and opacity using three additional 1D mappings for $f(x)$, $f''(x)$, and $O_{std}(f''(x))$ [S1]. The final mapping is:
    $$ f(x) \times O_{std}(f''(x)) \mapsto [0, 1] \quad \text{or} \quad f''(x) \times O_{std}(f''(x)) \mapsto [0, 1] $$
    [S1].

**Pseudocode**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def compute_bsatf(volume_f, visibility_func, num_directions, samples_per_ray):
    """
    Computes the Boundary-Structure-Aware Transfer Function features.
    
    Args:
        volume_f: 3D numpy array, input scalar field f(x).
        visibility_func: Function M(p, d) returning visibility for voxel p and field d.
        num_directions: H, number of directions for standard deviation.
        samples_per_ray: T, number of samples along each direction.
        
    Returns:
        divergence_field: 3D numpy array, f''(x).
        std_occlusion_field: 3D numpy array, O_std(f''(x)).
    """
    import numpy as np
    from scipy.ndimage import laplace
    
    # Step 1: Compute Divergence (Laplacian) of f(x)
    # f''(x) = div(grad(f(x)))
    divergence_field = laplace(volume_f)
    
    # Step 2: Use divergence_field as input d for occlusion
    d = divergence_field
    
    # Step 3 & 4 & 5: Compute Standard Deviation of Ambient Occlusion
    # Initialize field to store directional occlusions
    # Shape: (H, V, V, V) where V is volume dimensions
    directional_occlusions = np.zeros((num_directions, *volume_f.shape))
    
    # Generate H directions uniformly distributed on sphere
    # [G] Implementation detail: Use spherical coordinates or fibonacci sphere sampling
    directions = generate_uniform_sphere_samples(num_directions)
    
    for i, omega in enumerate(directions):
        # Compute directional occlusion for all voxels along direction omega
        # A(p, omega, d) = sum_{t=0}^{T} M(p + t*omega, d)
        # [G] Ray casting implementation: For each voxel p, cast ray in direction omega
        # and sum visibility values. Handle boundary conditions by clamping or zero-padding.
        directional_occlusions[i] = compute_directional_occlusion_field(
            volume=d, 
            direction=omega, 
            samples=samples_per_ray, 
            visibility_func=visibility_func
        )
        
    # Compute Standard Deviation across the H directions
    # O_std(p, d) = sqrt( mean(A^2) - (mean(A))^2 )
    mean_squared = np.mean(np.square(directional_occlusions), axis=0)
    squared_mean = np.square(np.mean(directional_occlusions, axis=0))
    
    std_occlusion_field = np.sqrt(mean_squared - squared_mean)
    
    return divergence_field, std_occlusion_field

def compute_directional_occlusion_field(volume, direction, samples, visibility_func):
    """
    Computes the directional occlusion field A(p, omega, d) for a given direction.
    """
    result = np.zeros_like(volume)
    # [G] Iterate over all voxels p in volume
    # For each p, cast 'samples' steps along 'direction'
    # Sum visibility_func(p + t*direction, volume) for t in 0..samples-1
    # Store sum in result[p]
    # Note: Efficient implementation uses vectorized ray casting or GPU kernels.
    # Here is a conceptual loop for clarity:
    for p in np.ndindex(volume.shape):
        occlusion_sum = 0.0
        for t in range(samples):
            # Calculate ray position
            ray_pos = p + t * direction
            # Clamp to volume bounds
            ray_pos_clamped = np.clip(ray_pos, 0, np.array(volume.shape) - 1)
            # Get visibility
            val = visibility_func(ray_pos_clamped, volume)
            occlusion_sum += val
        result[p] = occlusion_sum
    return result

def generate_uniform_sphere_samples(n):
    """
    Generates n unit vectors uniformly distributed on a sphere.
    [G] Standard implementation using spherical coordinates or golden spiral.
    """
    # Example: Golden Spiral method
    indices = np.arange(0, n, dtype=float) + 0.5
    phi = np.arccos(1 - 2 * indices / n)
    theta = np.pi * (1 + np.sqrt(5)) * indices
    x = np.sin(phi) * np.cos(theta)
    y = np.sin(phi) * np.sin(theta)
    z = np.cos(phi)
    return np.vstack((x, y, z)).T
```

**Edge Cases and Pitfalls**
*   **Simple Volumes:** In volumes with only two objects, using $f''(x)$ as the input field $\vec{d}$ may result in most values being close to zero, making separation difficult. In such cases, using $f(x)$ as $\vec{d}$ is recommended [S1].
*   **Non-Gaussian Boundaries:** Existing boundary-based methods like Kindlmann et al.'s fail on non-Gaussian boundaries or varying intensities. BSATF handles this by combining $f''(x)$ with $O_{std}$ [S1].
*   **Identical Surroundings:** Standard ambient occlusion fails to distinguish objects with the same intensity and same surroundings (e.g., objects at different locations) because it sums all voxel values, resulting in identical occlusion values. BSATF uses standard deviation to capture variation in directional occlusions, solving this [S1].
*   **Continuous Intensity Changes:** Objects with continuously changing internal intensities (e.g., marrow in MRI) are hard to extract using intensity alone. The divergence $f''(x)$ suppresses these continuous changes, allowing $O_{std}$ to distinguish them based on global structure [S1].

## Operating characteristics
*   **Complexity:** The method requires computing the Laplacian ($O(N)$) and ambient occlusion for $H$ directions and $T$ samples per voxel ($O(N \cdot H \cdot T)$). This is computationally expensive compared to local gradient methods.
*   **Memory:** Requires storage for the input field, divergence field, and directional occlusion fields for $H$ directions.
*   **Strengths:**
    *   Separates objects with similar intensities but different locations or surroundings [S1].
    *   Handles objects with continuously changing internal intensities (non-homogeneous interiors) [S1].
    *   Distinguishes objects with same size and distance that overlap in standard occlusion spectra [S1].
*   **Limitations:**
    *   Computationally intensive due to global occlusion calculation over many directions [S1].
    *   Requires careful selection of visibility function $M(p, \vec{d})$ [S1].
*   **When to Use:** Ideal for medical imaging (e.g., separating marrow from muscle/bone in MRI) and engineering data (e.g., extracting holes in combustion simulations) where structures have similar intensities but distinct global contexts or internal variations [S1].
*   **When Not to Use:** Simple volumes with few objects where $f(x)$ suffices, or when computational resources are severely limited [S1].

## Background
*   **Origin:** Proposed by Lina Yu and Hongfeng Yu at the University of Nebraska-Lincoln [S1].
*   **Publication:** Presented at the Symposium on Visualization (SA '17) in Bangkok, Thailand, November 27-30, 2017 [S1].
*   **Development:** The method advances classification by bridging boundary-based and structure-based methods. It addresses limitations of Correa et al.'s ambient occlusion method (which uses summation $O(f(x))$ and fails for objects with same intensity/size/distance) and Kindlmann et al.'s derivative-based method (which fails on non-Gaussian boundaries) by using the standard deviation of ambient occlusion on the divergence field $O_{std}(f''(x))$ [S1].
*   **Usage:** Applied to MRI scans of feet (separating marrow, bone, skin) and CT scans of engine data (separating stick-shape holes) and combustion simulations (extracting holes on flame isosurfaces) [S1].

## Sources
[S1] Boundary-structure-aware transfer functions for volume classification, https://doi.org/10.1145/3139295.3139306, fetched 2026-10-31
