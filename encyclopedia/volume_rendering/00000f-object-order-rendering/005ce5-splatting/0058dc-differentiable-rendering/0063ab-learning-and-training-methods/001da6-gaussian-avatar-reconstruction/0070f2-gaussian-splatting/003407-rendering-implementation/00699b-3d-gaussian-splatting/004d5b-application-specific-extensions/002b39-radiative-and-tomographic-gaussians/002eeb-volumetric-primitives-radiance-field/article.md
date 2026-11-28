# Volumetric Primitives Radiance Field

Also called: VPRF

## Overview
Volumetric Primitives Radiance Field (VPRF) is a rendering integrator variant designed for 3D Gaussian Splatting-like assets, optimized specifically for radiance field optimization and rendering tasks [S1, unofficial]. It shares the core volumetric primitive structure of the Volumetric Primitives Path Tracer (VPPT) but diverges by simplifying the light transport model to consider only primary rays from the camera [S1, unofficial][S2, unofficial]. Unlike VPPT, which targets general volumetric scattering media, VPRF is tailored for reconstructing and rendering emissive radiance fields [S1, unofficial].

## How it works
VPRF renders scenes composed of 3D Gaussian kernels by accumulating emission along primary camera rays [S2, unofficial]. It relies on the closed-form integration of spherical harmonics within the segments defined by the intersection of rays and volumetric primitives [S2, unofficial].

**Inputs and Outputs:**
*   **Input:** A set of $M$ 3D Gaussian primitives $\mathcal{G} = \{G_i\}_{i=1}^M$, where each primitive is defined by position $\mathbf{p}_i$, rotation $\mathbf{R}_i$, and scale $\mathbf{S}_i$ [Radiative Gaussian Splatting](3040-radiative-gaussian-splatting.md). Each primitive carries view-independent scalar attributes or feature vectors representing emission [Radiative Gaussian Splatting](3040-radiative-gaussian-splatting.md). Camera parameters defining primary rays.
*   **Output:** Pixel colors (radiance) for the image plane.

**Data Structures:**
The primitive representation follows the standard 3DGS layout:
*   $\mathbf{p}_i \in \mathbb{R}^3$: Position.
*   $\mathbf{R}_i \in \mathbb{R}^{3 \times 3}$: Rotation matrix.
*   $\mathbf{S}_i \in \mathbb{R}^{3 \times 3}$: Scale matrix.
*   Covariance: $\mathbf{\Sigma}_i = \mathbf{R}_i \mathbf{S}_i \mathbf{S}_i^\top \mathbf{R}_i^\top$ [Radiative Gaussian Splatting](3040-radiative-gaussian-splatting.md).
*   Attributes: View-independent emission coefficients (typically represented via Spherical Harmonics or simple scalars per band/channel) [Radiative Gaussian Splatting](3040-radiative-gaussian-splatting.md).

**Algorithm Steps:**
1.  **Ray Generation:** Generate primary rays $\mathbf{r}(t) = \mathbf{o} + t\mathbf{d}$ from the camera center $\mathbf{o}$ through each pixel direction $\mathbf{d}$ [S2, unofficial].
2.  **Intersection:** For each ray, determine intersections with the 3D Gaussian primitives [S2, unofficial]. The closed-form nature of the Gaussian volume allows for exact calculation of entry and exit points (segments) without probabilistic sampling [S2, unofficial].
3.  **Segmentation:** Split the ray into segments based on the intersection intervals with the primitives [S2, unofficial]. The order of primitives along the ray is strictly enforced to avoid ordering artifacts [S2, unofficial].
4.  **Emission Accumulation:** For each segment $k$ within a primitive $i$, compute the accumulated emission [S2, unofficial]. This is done by integrating the primitive's emission function (modeled via Spherical Harmonics) over the segment length [S2, unofficial]. The integration leverages the analytical properties of the kernel to provide a closed-form solution, eliminating noise associated with stochastic methods [S2, unofficial].
5.  **Compositing:** Composite the accumulated emissions from all segments along the ray to produce the final pixel color [S2, unofficial]. Since VPRF focuses on primary rays and emission, absorption and scattering between primitives are typically simplified or omitted compared to full path tracing [S1, unofficial][S2, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import numpy as np
import math

def render_vprf(primitives, camera):
    """
    Render a scene using the VPRF integrator.
    primitives: List of dicts with keys: p, R, S, emission
    camera: Object with attributes height, width and method get_primary_ray(x, y)
    """
    image = np.zeros((camera.height, camera.width, 3))
    
    for y in range(camera.height):
        for x in range(camera.width):
            ray_origin, ray_dir = camera.get_primary_ray(x, y)
            total_radiance = np.zeros(3)
            
            # 1. Find intersections and sort by distance
            intersections = []
            for prim in primitives:
                t_start, t_end = intersect_gaussian(ray_origin, ray_dir, prim)
                if t_start is not None:
                    intersections.append((t_start, t_end, prim))
            
            # Sort intersections by t_start to ensure correct ordering
            intersections.sort(key=lambda item: item[0])
            
            # 2. Accumulate emission along segments
            for t_start, t_end, prim in intersections:
                segment_radiance = integrate_emission_segment(ray_origin, ray_dir, t_start, t_end, prim)
                total_radiance += segment_radiance
            
            image[y, x] = total_radiance
            
    return image

def intersect_gaussian(ray_origin, ray_dir, prim):
    """Returns (t_start, t_end) or (None, None) if no intersection."""
    R_T = prim['R'].T
    S_inv = np.linalg.inv(prim['S'])
    o_local = S_inv @ R_T @ (ray_origin - prim['p'])
    d_local = S_inv @ R_T @ ray_dir
    
    # Ray-ellipsoid intersection quadratic: a*t^2 + b*t + c = 0
    a = np.dot(d_local, d_local)
    if a < 1e-8: return None, None
    b = 2.0 * np.dot(o_local, d_local)
    c = np.dot(o_local, o_local) - 1.0
    disc = b*b - 4*a*c
    
    if disc < 0: return None, None
    sqrt_disc = math.sqrt(disc)
    t1 = (-b - sqrt_disc) / (2*a)
    t2 = (-b + sqrt_disc) / (2*a)
    return t1, t2

def integrate_emission_segment(ray_origin, ray_dir, t_start, t_end, prim):
    """
    Computes the radiance emitted by a Gaussian primitive along a ray segment.
    Uses closed-form integration of the 1D Gaussian density weighted by emission.
    """
    R_T = prim['R'].T
    S_inv = np.linalg.inv(prim['S'])
    o_local = S_inv @ R_T @ (ray_origin - prim['p'])
    d_local = S_inv @ R_T @ ray_dir
    
    # Quadratic coefficients for |o_local + t*d_local|^2
    A = np.dot(d_local, d_local)
    B = 2.0 * np.dot(o_local, d_local)
    C = np.dot(o_local, o_local)
    
    if A < 1e-8: return np.zeros(3)
    
    # Complete the square: A(t - t0)^2 + D
    t0 = -B / (2.0 * A)
    D = C - (B*B) / (4.0 * A)
    
    # Closed-form integral of exp(-0.5*(A*t^2 + B*t + C)) using erf [G]
    integral = math.sqrt(math.pi / (2.0 * A)) * math.exp(-0.5 * D) * (
        math.erf(math.sqrt(A / 2.0) * (t_end - t0)) -
        math.erf(math.sqrt(A / 2.0) * (t_start - t0))
    )
    
    emission = prim.get('emission', np.ones(3))
    return emission * integral
```

**Parameters:**
*   **Spherical Harmonics Degree:** Determines the view-independent color fidelity [S2, unofficial].
*   **Ray Depth:** Fixed at 1 (primary rays only) [S2, unofficial].

**Edge Cases:**
*   **Degenerate Gaussians:** If scale matrices result in zero volume, intersection calculations may fail [G]. Guard against zero-scale primitives by clamping scale diagonal elements to a minimum epsilon [G].
*   **Precision:** Not given in the sources.

## Subtopics

### Volumetric Primitives Path Tracer (VPPT)
A sibling integrator that handles general volumetric scattering media, including multiple scattering and absorption, unlike VPRF's primary-ray-only emission model [S1, unofficial]. Choose VPPT for physically accurate volumetric light transport; choose VPRF for efficient radiance field optimization.

### Radiative Gaussian Splatting
The foundational method that replaces RGB opacity/color with view-independent scalar attributes or feature vectors, enabling the emission-based rendering used by VPRF [Radiative Gaussian Splatting](3040-radiative-gaussian-splatting.md).

## Operating characteristics
*   **Time Complexity:** Linear with respect to the number of primitives $M$ and pixels, assuming efficient spatial acceleration structures for intersection testing [G]. The closed-form integration avoids iterative sampling, reducing per-ray cost compared to stochastic volumetric rendering [S2, unofficial].
*   **Memory Complexity:** Linear with respect to $M$, storing Gaussian attributes and SH coefficients [G]. Significantly more memory-efficient than voxel grids for complex volumes [S2, unofficial].
*   **Strengths:** Zero noise due to analytical integration; fast rendering of emissive media; compatible with radiance field optimization pipelines; avoids ordering artifacts through strict ray ordering [S2, unofficial].
*   **Limitations:** Limited to primary rays and emission; does not natively simulate complex scattering or global illumination within the integrator itself, though the closed-form capability can be incorporated into external path tracers [S1, unofficial][S2, unofficial].
*   **When to use:** Radiance field optimization, rendering 3D Gaussian Splatting assets, and scenarios requiring noise-free emissive volume rendering [S2, unofficial].
*   **When not to use:** Scenes requiring multiple scattering, complex absorption, or full path-traced global illumination without external integration [S1, unofficial][S2, unofficial].

## Background
VPRF was introduced as part of the "Volumetric Primitives" framework by Facebook Research, designed to render 3D Gaussian Splatting-like assets efficiently [S1, unofficial]. It builds upon the mathematical foundation of representing volumes with compact kernels (Gaussian or Epanechnikov), leveraging their analytical properties for integration [S2, unofficial]. The method is particularly relevant for the reconstruction and rendering of radiance fields from image collections [S2, unofficial].

## Sources
[S1] facebookresearch/volumetric_primitives, https://github.com/facebookresearch/volumetric_primitives, fetched 2026-10-26 (unofficial)
[S2] Don’t Splat your Gaussians: Volumetric Ray-Traced Primitives for Modeling and Rendering Scattering and Emissive Media (2025), https://qiita.com/matumu20/items/67b33681dc97ccbcc849, fetched 2026-10-27 (unofficial)
