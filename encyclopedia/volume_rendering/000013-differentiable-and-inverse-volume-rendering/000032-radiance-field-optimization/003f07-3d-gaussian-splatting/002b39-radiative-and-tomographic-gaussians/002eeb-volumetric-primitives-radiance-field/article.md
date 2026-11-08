# Volumetric Primitives Radiance Field


## Overview
Volumetric Primitives Radiance Field (VPRF) is a rendering integrator designed for 3D Gaussian Splatting-like assets, optimized specifically for radiance field optimization and rendering tasks [S1, unofficial]. It shares the core volumetric primitive structure of the Volumetric Primitives Path Tracer (VPPT) but diverges by simplifying the radiative transfer model to consider only primary rays from the camera and accumulating emission based on spherical harmonics within segments [S2, unofficial]. Unlike VPPT, which targets general volumetric scattering media, VPRF is tailored for reconstructing and rendering radiance fields from photographic data [S1, unofficial].

## How it works
**Inputs and Outputs:**
*   **Input:** A set of 3D Gaussian primitives (position, covariance, spherical harmonic coefficients for emission) and camera parameters.
*   **Output:** A 2D image representing the accumulated emission along primary rays.

**Data Structures:**
The scene is represented by a list of $M$ volumetric primitives. Each primitive $i$ contains:
*   $\mathbf{p}_i \in \mathbb{R}^3$: Position.
*   $\mathbf{\Sigma}_i \in \mathbb{R}^{3 \times 3}$: Covariance matrix (derived from rotation and scale).
*   $\mathbf{c}_i$: Spherical harmonic coefficients defining the view-dependent emission intensity.
*   $\alpha_i$: A scalar opacity or density parameter controlling the magnitude of emission/absorption.

**Rendering Pipeline:**
1.  **Ray Generation:** Generate primary rays $\mathbf{r}(t) = \mathbf{o} + t\mathbf{d}$ from the camera origin $\mathbf{o}$ through each pixel direction $\mathbf{d}$ [S2, unofficial].
2.  **Ray-Primitive Intersection:** For each ray, determine the intersection intervals (segments) with the 3D Gaussians. Since Gaussians are unbounded, this involves solving for the parameter $t$ where the ray distance to the Gaussian center satisfies the quadratic form defined by the inverse covariance [S2, unofficial].
3.  **Segmentation:** Sort the intersection intervals along the ray. The ray is divided into segments where the density/emission contribution is integrated [S2, unofficial].
4.  **Emission Accumulation:** For each segment, compute the accumulated emission. The contribution of a Gaussian is integrated over the segment length. The emission color is evaluated using the spherical harmonics based on the view direction $\mathbf{d}$ [S2, unofficial].
5.  **Compositing:** Combine the contributions from all segments using alpha compositing. Since VPRF focuses on emission (and potentially absorption), the final color $C$ is computed as:
    $$ C = \sum_{i} T_i \cdot E_i $$
    where $T_i$ is the transmittance up to segment $i$, and $E_i$ is the emitted radiance from segment $i$ [S2, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def render_vprf(primitives, camera):
    """
    Render a scene using the VPRF integrator.
    primitives: List of Gaussian objects with pos, cov, sh_coeffs, opacity.
    camera: Camera object with origin, direction, and image dimensions.
    """
    image = zeros(camera.height, camera.width, 3)
    
    for y in range(camera.height):
        for x in range(camera.width):
            # 1. Generate primary ray
            ray_origin = camera.origin
            ray_dir = camera.get_direction(x, y)
            
            # 2. Find intersections with all primitives
            intersections = []
            for prim in primitives:
                # Solve quadratic equation for ray-Gaussian intersection
                # |r(t) - p|^2_Sigma = 1
                # This yields a range [t_start, t_end] if it intersects
                t_start, t_end = prim.intersect(ray_origin, ray_dir)
                if t_start is not None:
                    intersections.append((t_start, t_end, prim))
            
            # 3. Sort intersections by t_start
            intersections.sort(key=lambda x: x[0])
            
            # 4. Accumulate emission
            current_transmittance = 1.0
            accumulated_color = zeros(3)
            
            for t_start, t_end, prim in intersections:
                # 5. Integrate emission over the segment
                # Simplified: assume constant emission over segment for primary ray
                # Emission is evaluated using Spherical Harmonics
                view_dir = ray_dir
                emission_color = evaluate_sh(prim.sh_coeffs, view_dir)
                
                # Alpha for this segment based on primitive opacity and segment length
                # [G] Exact integration formula for Gaussian along ray is complex;
                # using a simplified approximation for demonstration.
                segment_length = t_end - t_start
                alpha = prim.opacity * gaussian_density_at_center(prim, ray_origin, ray_dir, segment_length)
                
                # 6. Composite
                contribution = current_transmittance * alpha * emission_color
                accumulated_color += contribution
                current_transmittance *= (1.0 - alpha)
                
                # Early exit if fully opaque
                if current_transmittance < 1e-6:
                    break
            
            image[y, x] = accumulated_color
            
    return image
```

**Parameters:**
*   **Spherical Harmonics Degree:** Determines the view-dependency of the emission. Higher degrees allow for more complex lighting but increase memory [S2, unofficial].
*   **Opacity/Density:** Controls the strength of the primitive's contribution [S2, unofficial].

**Edge Cases:**
*   **Degenerate Gaussians:** If the covariance matrix is singular, intersection calculations may fail. [G] Add a small epsilon to the diagonal of the covariance matrix.
*   **Deep Scenes:** Sorting intersections can be expensive. [G] Use a spatial acceleration structure (e.g., BVH) to cull primitives not intersecting the frustum.

## Subtopics

### Volumetric Primitives Path Tracer (VPPT)
A general-purpose integrator for volumetric scattering media using the same primitive representation. It handles multiple scattering and complex light paths, unlike VPRF which is limited to primary rays [S1, unofficial]. Choose VPPT for physically accurate volumetric rendering; choose VPRF for radiance field optimization.

### 3D Gaussian Splatting
The foundational method that VPRF extends. 3DGS uses alpha compositing of 2D projected Gaussians for fast rasterization. VPRF uses volumetric primitives for more accurate physical modeling [S1, unofficial].

## Operating characteristics
*   **Time Complexity:** $O(N \cdot M)$ for $N$ pixels and $M$ primitives, assuming no acceleration structure. With a BVH, it approaches $O(N \log M)$ for intersection testing.
*   **Memory Complexity:** $O(M)$ to store the primitives.
*   **Strengths:** Provides closed-form solutions for ray-primitive intersections, reducing noise compared to stochastic methods. Suitable for differentiable rendering and optimization [S2, unofficial].
*   **Limitations:** Limited to primary rays, making it unsuitable for complex global illumination effects like multiple scattering. Performance degrades with very high numbers of primitives without acceleration.
*   **When to use:** Radiance field optimization, novel view synthesis from photos, and scenarios requiring differentiable rendering of volumetric data.
*   **When not to use:** Real-time rendering requiring high frame rates with millions of primitives, or scenes requiring complex light transport.

## Background
VPRF is part of the "Volumetric Primitives" framework introduced by Facebook Research [S1, unofficial]. It was developed to address the limitations of standard rasterization-based 3D Gaussian Splatting in physically based rendering contexts. By using volumetric primitives, it enables more accurate modeling of light transport for radiance field applications [S1, unofficial]. The method leverages the mathematical properties of Gaussian kernels to allow for exact integration along rays, a key insight from the broader Volumetric Primitives research [S2, unofficial].

## Sources
[S1] facebookresearch/volumetric_primitives, https://github.com/facebookresearch/volumetric_primitives, fetched 2026-10-26 (unofficial)
[S2] Don’t Splat your Gaussians: Volumetric Ray-Traced Primitives for Modeling and Rendering Scattering and Emissive Media (2025), https://qiita.com/matumu20/items/67b33681dc97ccbcc849, fetched 2026-10-27 (unofficial)
