# Flipped Reflection Ray Masking

Also called: Flipped Reflection Rays

## Overview
Flipped Reflection Ray Masking is a regularization method for few-shot novel view synthesis that generates additional training rays from input viewing directions and estimated surface normals, casting them toward the identical target pixels as the original rays [S1]. It filters out ineffective rays based on the angle between the viewing direction and the normal to stabilize training and reduce floating artifacts [S1].

## How it works
**Inputs and Outputs:**
- **Inputs:** Original input ray directions ($\mathbf{d}_{in}$), estimated surface normal vectors ($\mathbf{n}$), and a threshold angle parameter ($\tau$) [S1].
- **Outputs:** A filtered set of flipped reflection rays ($\mathbf{d}_{flip}$) used for training supervision [S1].

**Data Structures:**
The method operates on standard 3D vector arrays for ray directions and surface normals [G].

**Steps:**
1. Derive the flipped reflection rays from the original input ray directions and the estimated surface normals [S1].
2. Cast the derived rays toward the identical target pixels as the original input rays [S1].
3. Compute $\theta$, the angle between the input viewing direction $\mathbf{d}_{in}$ and the surface normal vector $\mathbf{n}$ at each pixel [S1].
4. If $\theta > \tau$, discard the corresponding flipped reflection ray; otherwise, retain it for training supervision [S1].

**Formulas:**
The flipped reflection direction is computed as:
$$ \mathbf{d}_{flip} = 2(\mathbf{d}_{in} \cdot \mathbf{n})\mathbf{n} - \mathbf{d}_{in} $$
where $\mathbf{d}_{in}$ is the original viewing direction and $\mathbf{n}$ is the surface normal [S1]. The angle $\theta$ is calculated as:
$$ \theta = \arccos\left( \frac{\mathbf{d}_{in} \cdot \mathbf{n}}{\|\mathbf{d}_{in}\| \|\mathbf{n}\|} \right) $$
The term "flipped" refers to the reflected ray having an opposite direction relative to the camera, effectively pointing from the object surface back toward the camera's general hemisphere [S1].

**Parameters:**
- $\tau$ (tau): Threshold angle in degrees for ray masking. The optimal value is $90^\circ$ [S1].

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
        surface_normals (np.ndarray): Array of shape (N, 3) representing estimated surface normals.
        threshold_degrees (float): The threshold angle tau in degrees. Default 90.0.

    Returns:
        np.ndarray: Array of shape (N, 3) containing the flipped reflection rays.
                    Pixels where the ray was masked are set to zero vectors.
    """
    # Normalize input vectors for accurate angle calculation [G]
    d_in = input_ray_dirs / np.linalg.norm(input_ray_dirs, axis=1, keepdims=True)
    n = surface_normals / np.linalg.norm(surface_normals, axis=1, keepdims=True)

    # Step 1: Compute the flipped reflection direction
    dot_product = np.sum(d_in * n, axis=1)
    d_flip = 2.0 * dot_product[:, np.newaxis] * n - d_in

    # Step 2: Compute the angle theta between input direction and normal
    cos_theta = np.clip(dot_product, -1.0, 1.0)
    theta_degrees = np.degrees(np.arccos(cos_theta))

    # Step 3: Apply masking
    # If theta > tau, discard the ray; otherwise, retain it [S1]
    mask = theta_degrees <= threshold_degrees
    d_flip_masked = np.where(mask[:, np.newaxis], d_flip, 0.0)

    return d_flip_masked
```

**Edge Cases and Pitfalls:**
- **Degenerate Normals:** If surface normals are zero vectors or poorly estimated (common in early training phases), the angle calculation becomes unstable. The masking strategy effectively filters these invalid additional rays at the initial training phase, leading to high-quality supervision and stabilizing training [S1].
- **Precision:** Ensure dot products are clipped to $[-1, 1]$ before `arccos` to avoid numerical domain errors [G].

## Subtopics

### Uncertainty-aware Emptiness Loss
A complementary loss function used with flipped rays to reduce floating artifacts based on uncertainty [S1].

### Bottleneck Feature Consistency Loss
Ensures consistency between bottleneck features of original and flipped rays [S1].

## Operating characteristics
- **Time Complexity:** Not given in the sources. The method adds a parallel branch of ray supervision without increasing the number of ray march steps [G].
- **Memory Complexity:** Not given in the sources. Requires storing additional ray directions and normals for the flipped rays during training [G].
- **Performance:** On the Realistic Synthetic 360° dataset, the $90^\circ$ masking strategy yields the highest PSNR (19.55), SSIM (0.767), and lowest LPIPS (0.180) and Average Error (0.101) among masking variants [S1].
- **Strengths:** Stabilizes training in few-shot scenarios by filtering invalid rays at the initial phase and reducing floating artifacts [S1].
- **Limitations:** Using too strict a threshold (e.g., $\tau = 30^\circ$ or $60^\circ$) degrades quality by filtering too many rays, reducing supervision [S1]. Using no masking yields inferior results due to noise from ineffective rays [S1].
- **When to use:** Few-shot novel view synthesis where surface normal estimation is available or learnable [S1].
- **When not to use:** Dense-view scenarios where standard supervision is sufficient, or when normal estimation is unreliable [G].

## Background
Flipped Reflection Ray Masking was introduced in the FlipNeRF paper in 2023 [S1]. It is built upon MixNeRF, leveraging a mixture model framework [S1]. The method addresses the limited input views in novel view synthesis by using flipped reflection rays as effective additional training resources [S1]. It is particularly relevant for few-shot learning where accurate depth and normal estimation are critical [S1].

## Sources
[S1] FlipNeRF: Flipped Reflection Rays for Few-shot Novel View Synthesis, https://doi.org/10.1109/iccv51070.2023.02092, fetched 2026-10-29
