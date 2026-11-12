# Volumetric Primitives Radiance Field


## Overview
Volumetric Primitives Radiance Field (VPRF) is a rendering integrator designed for 3D Gaussian Splatting-like assets, optimized specifically for radiance field optimization and rendering tasks [S1, unofficial]. It shares the core volumetric primitive structure of the Volumetric Primitives Path Tracer (VPPT) but diverges by simplifying the radiative transfer model to consider only primary rays from the camera and accumulating emission based on spherical harmonics within segments [S2, unofficial]. Unlike VPPT, which targets general volumetric scattering media, VPRF is tailored for reconstructing and rendering radiance fields from photographic data [S1, unofficial].

## How it works
**Inputs and Outputs:**
*   **Input:** A set of volumetric primitives (position, covariance, spherical harmonic coefficients for emission) and camera parameters [S1, unofficial][S2, unofficial].
*   **Output:** A 2D image representing the accumulated emission along primary rays [S2, unofficial].

**Data Structures:**
The scene is represented by a list of $M$ primitives. Each primitive $i$ contains:
*   $\mathbf{p}_i \in \mathbb{R}^3$: Position [S1, unofficial].
*   $\mathbf{\Sigma}_i \in \mathbb{R}^{3 \times 3}$: Covariance matrix (derived from rotation and scale) [S1, unofficial].
*   $\mathbf{c}_i$: Spherical harmonic coefficients defining the view-dependent emission intensity [S2, unofficial].
*   $\alpha_i$: A scalar opacity or density parameter controlling the magnitude of emission/absorption [G].

**Rendering Pipeline:**
1.  **Ray Generation:** Generate primary rays $\mathbf{r}(t) = \mathbf{o} + t\mathbf{d}$ from the camera origin $\mathbf{o}$ through each pixel direction $\mathbf{d}$ [S2, unofficial].
2.  **Ray-Primitive Intersection:** For each ray, determine the intersection intervals (segments) with the primitives. The method uses closed-form solutions for ray-primitive intersections to avoid stochastic noise [S2, unofficial].
3.  **Segmentation:** Sort the intersection intervals along the ray. The ray is divided into segments where the emission contribution is integrated [S2, unofficial].
4.  **Emission Accumulation:** For each segment, compute the accumulated emission. The emission color is evaluated using spherical harmonics based on the view direction $\mathbf{d}$ [S2, unofficial].
5.  **Compositing:** Combine the contributions from all segments using alpha compositing. The final color is computed by accumulating emitted radiance weighted by transmittance along the ray [G].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import math

def zeros(shape, dtype=float):
    if isinstance(shape, int):
        return [0.0] * shape
    return [zeros(s, dtype) for s in shape]

def evaluate_sh(coeffs, direction):
    # [G] Standard spherical harmonics evaluation; degree 0 is isotropic.
    return coeffs[0]

def gaussian_density_at_center(prim, origin, direction, segment_length):
    # [G] Approximation: density at center scaled by segment length.
    # Exact integration requires error functions; this is a standard simplification.
    return math.exp(-0.5 * segment_length)

def render_vprf(primitives, camera):
    image = zeros((camera.height, camera.width, 3))
    for y in range(camera.height):
        for x in range(camera.width):
            ray_origin = camera.origin
            ray_dir = camera.get_direction(x, y)
            intersections = []
            for prim in primitives:
                t_start, t_end = prim.intersect(ray_origin, ray_dir)
                if t_start is not None:
                    intersections.append((t_start, t_end, prim))
            intersections.sort(key=lambda x: x[0])
            current_transmittance = 1.0
            accumulated_color = zeros(3)
            for t_start, t_end, prim in intersections:
                view_dir = ray_dir
                emission_color = evaluate_sh(prim.sh_coeffs, view_dir)
                segment_length = t_end - t_start
                alpha = prim.opacity * gaussian_density_at_center(prim, ray_origin, ray_dir, segment_length)
                contribution = current_transmittance * alpha * emission_color
                accumulated_color = [accumulated_color[i] + contribution[i] for i in range(3)]
                current_transmittance *= (1.0 - alpha)
                if current_transmittance < 1e-6:
                    break
            image[y][x] = accumulated_color
    return image
```

**Parameters:**
*   **Spherical Harmonics Degree:** Determines the view-dependency of the emission [S2, unofficial].
*   **Opacity/Density:** Controls the strength of the primitive's contribution [G].

**Edge Cases:**
*   **Degenerate Gaussians:** If the covariance matrix is singular, intersection calculations may fail. [G] Add a small epsilon to the diagonal of the covariance matrix.
*   **Deep Scenes:** Sorting intersections can be expensive. [G] Use a spatial acceleration structure (e.g., BVH) to cull primitives not intersecting the frustum.

## Subtopics

### Volumetric Primitives Path Tracer (VPPT)
A general-purpose integrator for volumetric scattering media using the same primitive representation [S1, unofficial]. It handles multiple scattering and complex light paths, unlike VPRF which is limited to primary rays [S2, unofficial]. Choose VPPT for physically accurate volumetric rendering; choose VPRF for radiance field optimization [S1, unofficial].

### 3D Gaussian Splatting
The foundational method that VPRF extends. 3DGS uses alpha compositing of 2D projected Gaussians for fast rasterization [G]. VPRF uses volumetric primitives for more accurate physical modeling [S1, unofficial].

## Operating characteristics
*   **Time Complexity:** $O(N \cdot M)$ for $N$ pixels and $M$ primitives, assuming no acceleration structure [G]. With a BVH, it approaches $O(N \log M)$ for intersection testing [G].
*   **Memory Complexity:** $O(M)$ to store the primitives [G].
*   **Strengths:** Provides closed-form solutions for ray-primitive intersections, reducing noise compared to stochastic methods [S2, unofficial]. Suitable for differentiable rendering and optimization [S2, unofficial].
*   **Limitations:** Limited to primary rays, making it unsuitable for complex global illumination effects like multiple scattering [S2, unofficial]. Performance degrades with very high numbers of primitives without acceleration [G].
*   **When to use:** Radiance field optimization, novel view synthesis from photos, and scenarios requiring differentiable rendering of volumetric data [S2, unofficial].
*   **When not to use:** Real-time rendering requiring high frame rates with millions of primitives, or scenes requiring complex light transport [G].

## Background
VPRF is part of the "Volumetric Primitives" framework introduced by Facebook Research [S1, unofficial]. It was developed to address the limitations of standard rasterization-based 3D Gaussian Splatting in physically based rendering contexts [S1, unofficial]. By using volumetric primitives, it enables more accurate modeling of light transport for radiance field applications [S2, unofficial]. The method leverages the mathematical properties of volumetric kernels to allow for exact integration along rays, a key insight from the broader Volumetric Primitives research [S2, unofficial].

## Sources
[S1] facebookresearch/volumetric_primitives, https://github.com/facebookresearch/volumetric_primitives, fetched 2026-10-26 (unofficial)
[S2] Don’t Splat your Gaussians: Volumetric Ray-Traced Primitives for Modeling and Rendering Scattering and Emissive Media (2025), https://qiita.com/matumu20/items/67b33681dc97ccbcc849, fetched 2026-10-27 (unofficial)
