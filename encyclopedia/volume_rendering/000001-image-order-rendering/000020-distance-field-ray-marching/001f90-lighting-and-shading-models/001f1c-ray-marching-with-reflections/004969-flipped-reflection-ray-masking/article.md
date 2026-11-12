# Flipped Reflection Ray Masking

Also called: Flipped Reflection Rays

## Overview
Flipped Reflection Ray Masking is a regularization technique for few-shot novel view synthesis that generates additional training rays by reflecting input viewing directions across estimated surface normals. It solves the problem of insufficient supervision in sparse-view scenarios by providing geometrically consistent auxiliary signals, while discarding rays that are geometrically inconsistent with the scene to prevent noise. It sets itself apart from sibling methods like [Nested Raymarching for Refraction](1957-nested-raymarching-for-refraction.md) by focusing on reflection-based regularization rather than refraction physics, and from [Screen-Space Reflections](9698-screen-space-reflections.md) by operating in the training data generation phase rather than post-processing.

## How it works
**Inputs and Outputs:**
- **Inputs:** Original input ray directions ($\mathbf{d}_{in}$), estimated surface normal vectors ($\mathbf{n}$) at the intersection points, and a threshold angle parameter ($\tau$).
- **Outputs:** A filtered set of "flipped" reflection rays ($\mathbf{d}_{flip}$) used for training supervision.

**Data Structures:**
The method operates on ray vectors and normal vectors. No specific complex data structures are required beyond standard vector storage. The masking logic relies on scalar angle computation.

**Steps:**
1.  **Estimate Normals:** For each pixel in the input views, estimate the surface normal vector $\mathbf{n}$ derived from the volume densities or depth maps. [S1]
2.  **Compute Flipped Direction:** Calculate the direction of the flipped reflection ray $\mathbf{d}_{flip}$ using the reflection formula:
    $$ \mathbf{d}_{flip} = 2(\mathbf{d}_{in} \cdot \mathbf{n})\mathbf{n} - \mathbf{d}_{in} $$
    where $\mathbf{d}_{in}$ is the original viewing direction (pointing from camera to scene) and $\mathbf{n}$ is the surface normal. The term "flipped" refers to the resulting ray having a direction opposite to the standard reflection expectation relative to the camera, effectively casting rays from the object surface back towards the camera's general hemisphere. [S1]
3.  **Calculate Angle:** Compute the angle $\theta$ between the original input viewing direction $\mathbf{d}_{in}$ and the estimated surface normal $\mathbf{n}$:
    $$ \theta = \arccos\left( \frac{\mathbf{d}_{in} \cdot \mathbf{n}}{\|\mathbf{d}_{in}\| \|\mathbf{n}\|} \right) $$
    Note: $\mathbf{d}_{in}$ and $\mathbf{n}$ should be normalized. [S1]
4.  **Masking Decision:** Compare $\theta$ against the threshold $\tau$:
    - If $\theta > \tau$, discard the flipped reflection ray for this pixel.
    - If $\theta \le \tau$, retain the flipped reflection ray for training supervision. [S1]

**Parameters:**
- $\tau$ (tau): The threshold angle in degrees for ray masking. The optimal value is $90^\circ$. [S1]

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import math
import numpy as np

def compute_flipped_reflection_masking(input_ray_dirs, surface_normals, threshold_degrees=90.0):
    """
    Computes flipped reflection rays and masks ineffective ones based on the angle
    between the input viewing direction and the surface normal.

    Args:
        input_ray_dirs (np.ndarray): Array of shape (N, 3) representing input ray directions.
                                     These point from the camera to the scene.
        surface_normals (np.ndarray): Array of shape (N, 3) representing estimated surface normals.
        threshold_degrees (float): The threshold angle tau in degrees. Default 90.0.

    Returns:
        np.ndarray: Array of shape (N, 3) containing the flipped reflection rays.
                    Pixels where the ray was masked are set to zero vectors.
    """
    # Normalize input vectors to ensure accurate angle calculation
    # [G] Standard vector normalization
    d_in = input_ray_dirs / np.linalg.norm(input_ray_dirs, axis=1, keepdims=True)
    n = surface_normals / np.linalg.norm(surface_normals, axis=1, keepdims=True)

    # Step 1: Compute the flipped reflection direction
    # Formula: d_flip = 2 * (d_in . n) * n - d_in
    # Note: The paper describes deriving rays from input directions and normals.
    # Standard reflection formula reflects the incoming vector off the normal.
    dot_product = np.sum(d_in * n, axis=1)
    d_flip = 2.0 * dot_product[:, np.newaxis] * n - d_in

    # Step 2: Compute the angle theta between input direction and normal
    # theta = arccos( (d_in . n) / (||d_in|| ||n||) )
    # Since vectors are normalized, denominator is 1.
    cos_theta = np.clip(dot_product, -1.0, 1.0)
    theta_radians = np.arccos(cos_theta)
    theta_degrees = np.degrees(theta_radians)

    # Step 3: Apply masking
    # If theta > tau, discard the ray (set to zero or mask out)
    # Otherwise, retain it.
    mask = theta_degrees <= threshold_degrees

    # Apply mask: zero out discarded rays
    d_flip_masked = np.where(mask[:, np.newaxis], d_flip, 0.0)

    return d_flip_masked
```

**Edge Cases and Pitfalls:**
- **Degenerate Normals:** If surface normals are zero vectors or poorly estimated (common in early training phases), the angle calculation becomes unstable. The masking strategy helps stabilize initial training by filtering invalid rays. [S1]
- **Precision:** Ensure dot products are clipped to $[-1, 1]$ before `arccos` to avoid numerical domain errors. [G]

## Subtopics

### Uncertainty-aware Emptiness Loss
A complementary loss function used with flipped rays to reduce floating artifacts based on uncertainty. [S1]

### Bottleneck Feature Consistency Loss
Ensures consistency between bottleneck features of original and flipped rays. [S1]

## Operating characteristics
- **Time Complexity:** Adds a constant factor overhead per pixel for normal estimation, reflection vector calculation, and angle comparison. It does not increase the number of ray march steps but adds a parallel branch of ray supervision.
- **Memory Complexity:** Requires storing additional ray directions and normals for the flipped rays during training.
- **Performance:** On the Realistic Synthetic 360° dataset, using $\tau = 90^\circ$ yields the highest PSNR (19.55), SSIM (0.767), and lowest LPIPS (0.180) and Average Error (0.101) among masking variants. [S1]
- **Strengths:** Stabilizes training in few-shot scenarios by providing geometrically consistent supervision. Reduces floating artifacts. [S1]
- **Limitations:** Using too strict a threshold (e.g., $\tau = 30^\circ$ or $60^\circ$) degrades quality by filtering too many rays, reducing supervision. Using no masking yields inferior results due to noise from ineffective rays. [S1]
- **When to use:** Few-shot novel view synthesis where surface normal estimation is available or learnable.
- **When not to use:** Dense-view scenarios where standard supervision is sufficient, or when normal estimation is unreliable.

## Background
Flipped Reflection Ray Masking was introduced in the FlipNeRF paper in 2023. It is built upon MixNeRF, leveraging a mixture model framework. The method addresses the stumbling block of limited input views in novel view synthesis by using flipped reflection rays as effective additional training resources. It is particularly relevant for few-shot learning where accurate depth and normal estimation are critical. [S1]

## Sources
[S1] FlipNeRF: Flipped Reflection Rays for Few-shot Novel View Synthesis, https://doi.org/10.1109/iccv51070.2023.02092, fetched 2026-10-29
