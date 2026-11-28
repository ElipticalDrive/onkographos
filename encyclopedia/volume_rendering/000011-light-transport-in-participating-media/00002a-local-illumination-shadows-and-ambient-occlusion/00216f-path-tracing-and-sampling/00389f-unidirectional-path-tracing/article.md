# Unidirectional Path Tracing

Also called: Standard Path Tracing Lighting

## Overview
Unidirectional Path Tracing is a Monte Carlo method for estimating the rendering equation by tracing sequences of connected rays from the camera into the scene, handling surface scattering and volumetric interactions via stochastic perturbation [S2][S3, unofficial]. It synthesizes the image by collecting all light paths that connect a pixel on the camera sensor to an emitter, updating a throughput vector at each bounce to accumulate emitted or reflected light [S1, S3, unofficial]. It constructs paths unidirectionally from the camera, distinguishing it from bidirectional methods that sample from both ends, and from pure rasterization by explicitly simulating global illumination through recursive ray tracing [S2].

## How it works
**Inputs:** Camera ray (origin, direction), scene geometry (primitives, materials), medium stack (for volumetric regions), random number generator (sampler) [S3, unofficial].
**Outputs:** Radiance value $L$ for the pixel [S3, unofficial].

**Data Structures:**
*   `Ray`: Struct containing `origin` (Vec3f), `direction` (Vec3f), `throughput` (Vec3f, initialized to $(1,1,1)$), and `mediums` (stack of Medium pointers via pushMedium/popMedium) [S3, unofficial].
*   `SceneData`: Contains parameters including `rayBounces` (max depth), `sunDir` (vec3f), `sunIntensity` (f32), and `sunEnabled` (u32) [S1, unofficial].
*   `MATERIALS`: Constant array of size 5 where each material has `specular` (f32) and `emissive` (bool). Index 0: specular 0.0, not emissive. Index 1: specular 0.0, not emissive. Index 2: specular 0.0, emissive. Index 3: specular 0.9, not emissive. Index 4: specular 1.0, not emissive [S1, unofficial].

**Parameters:**
*   `maxDepth`: Maximum path length. Default value is 100 [S3, unofficial].
*   `rayBounces`: Controls the maximum number of ray bounces in `SceneData` [S1, unofficial].

**Algorithm:**
1.  Initialize radiance $L = 0$, throughput $T = (1,1,1)$, and depth $d = 0$ [S3, unofficial].
2.  Loop while $d < maxDepth$ [S3, unofficial].
3.  **Russian Roulette:** If $d > 0$, calculate survival probability $p = \min\left(\frac{T_r + T_g + T_b}{3}, 1.0\right)$ [S3, unofficial]. If `sampler.getNext1D()` $\ge p$, break loop. Otherwise, update $T \leftarrow T / p$ [S3, unofficial].
4.  **Intersection:** Intersect ray with scene primitives [G]. If no intersection, return $L$ [G].
5.  **Direct Emission Check:** If the hit primitive is an area light (emissive), add $T \cdot L_e$ to $L$ and break loop [S3, unofficial].
6.  **BSDF Sampling:** Sample the Bidirectional Scattering Distribution Function (BSDF) at the hit point [S3, unofficial].
    *   Compute incoming direction $\omega_i$ and outgoing direction $\omega_o$ (ray direction) [S3, unofficial].
    *   Evaluate BSDF $f(\omega_i, \omega_o)$ and PDF $p(\omega_o)$ [S3, unofficial].
    *   **Light Leak Guard:** Check asymmetry of BSDF using shading normals ($n_s$) and geometric normals ($n_g$). If $\omega_i \cdot n_g \cdot \omega_i \cdot n_s \le 0$ or $\omega_o \cdot n_g \cdot \omega_o \cdot n_s \le 0$, discard path [S3, unofficial].
    *   Update throughput: $T_{new} = T_{old} \cdot \frac{f(\omega_i, \omega_o) \cdot |\cos \theta|}{p(\omega_o)}$, where $\theta$ is the angle between the surface normal and the outgoing direction [S3, unofficial].
    *   Set flag `is_reflected_or_refracted = true` [S3, unofficial].
7.  **Medium Update:** Update the medium stack based on surface crossing using `updateMedium(ray, dir, info)` [S3, unofficial].
8.  **Ray Update:** Set ray origin to `surfaceInfo.position` and direction to the sampled $\omega_i$ [S3, unofficial].
9.  **Depth Increment:** If `is_scattered` or `is_reflected_or_refracted`, increment $d$ [S3, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
class Vec3:
    def __init__(self, x=0.0, y=0.0, z=0.0): self.x, self.y, self.z = x, y, z
    def __add__(self, o): return Vec3(self.x+o.x, self.y+o.y, self.z+o.z)
    def __sub__(self, o): return Vec3(self.x-o.x, self.y-o.y, self.z-o.z)
    def __mul__(self, s): return Vec3(self.x*s, self.y*s, self.z*s)
    def __truediv__(self, s): return Vec3(self.x/s, self.y/s, self.z/s)
    def dot(self, o): return self.x*o.x + self.y*o.y + self.z*o.z
    def length(self): return (self.x**2 + self.y**2 + self.z**2)**0.5
    def normalize(self): l = self.length(); return Vec3(self.x/l, self.y/l, self.z/l) if l > 0 else self

DIRECTION = {"FROM_CAMERA": 0}

def unidirectional_path_tracing(camera_ray, scene, sampler):
    radiance = Vec3(0.0, 0.0, 0.0)
    throughput = Vec3(1.0, 1.0, 1.0)
    depth = 0
    max_depth = 100  # Default value [S3, unofficial]

    ray = camera_ray
    ray.throughput = throughput

    while depth < max_depth:
        # Russian Roulette [S3, unofficial]
        if depth > 0:
            avg_throughput = (throughput.x + throughput.y + throughput.z) / 3.0
            probability = min(avg_throughput, 1.0)
            if sampler.get_next_1d() >= probability:
                break
            throughput = throughput / probability

        # Intersection [G]
        surface_info = scene.intersect(ray)
        if surface_info is None:
            break

        # Direct Emission Check [S3, unofficial]
        if surface_info.is_emissive:
            radiance = radiance + throughput * surface_info.emission
            break

        # BSDF Sampling [S3, unofficial]
        # Sample BSDF: returns outgoing direction (wo), PDF, and BSDF value (f)
        wo, pdf, f = surface_info.bsdf.sample(
            -ray.direction, surface_info, DIRECTION["FROM_CAMERA"], sampler
        )
        wi = -ray.direction  # Incoming direction to surface

        # Light Leak Guard [S3, unofficial]
        # Check asymmetry using shading normals (ns) and geometric normals (ng)
        wo_ns = wo.dot(surface_info.shading_normal)
        wo_ng = wo.dot(surface_info.geometric_normal)
        wi_ns = wi.dot(surface_info.shading_normal)
        wi_ng = wi.dot(surface_info.geometric_normal)
        if (wi_ng * wi_ns <= 0) or (wo_ng * wo_ns <= 0):
            break  # Discard path

        # Throughput Update [S3, unofficial]
        cos_theta = abs(wo.dot(surface_info.normal))
        throughput = throughput * (f * cos_theta) / pdf

        # Medium Update [G] # Source does not specify implementation details for updateMedium
        # update_medium(ray, wo, surface_info)

        # Ray Update [S3, unofficial]
        ray.origin = surface_info.position
        ray.direction = wo.normalize()
        ray.throughput = throughput

        # Depth Increment [S3, unofficial]
        depth += 1

    return radiance
```

## Subtopics

### [Volumetric Path Tracing](1863-volumetric-path-tracing.md)
A Monte Carlo method for simulating light transport in participating media, such as fog, smoke, and clouds, by modeling color-dependent absorption and scattering. It extends classical path tracing by allowing rays to enter volumetric regions, sample scattering events within the medium, and continue random walks after each scattering interaction to support multiple scattering. Unlike its sibling [Unidirectional Volume Boundary Estimator](3538-unidirectional-volume-boundary-estimator.md), which focuses on boundary estimation, or [Volumetric-Primitives Path Tracer](4107-volumetric-primitives-path-tracer.md), which uses primitives for acceleration, this method relies on delta tracking for volume sampling and does not perform explicit light sampling or handle surface scattering. It is primarily used for computing ground-truth results when debugging more sophisticated algorithms due to its simplicity and higher variance compared to light-sampling integrators.

### [Unidirectional Volume Boundary Estimator](3538-unidirectional-volume-boundary-estimator.md)
The Unidirectional Volume Boundary Estimator is a Monte Carlo method that samples volume boundary paths by building a main path from the detector to the light, then constructing side paths from main path vertices to the volume boundary and then to the light. It solves the problem of efficiently sampling paths where a scattering event occurs exactly on a volume boundary (interface). It sets itself apart from its sibling, the [Bidirectional Volume Boundary Estimator](4773-bidirectional-volume-boundary-estimator.md), by constructing the path unidirectionally: it samples the main path first and then the boundary vertex, whereas the bidirectional variant samples the boundary vertex first and connects it to both light and camera.

### [Volumetric-Primitives Path Tracer](4107-volumetric-primitives-path-tracer.md)
A forward rendering method for scattering and emissive media that models the scene as a collection of 3D kernel-based volumetric primitives and uses ray tracing to compute light transport, supporting multiple scattering via Next-Event Estimation. Unlike [Volumetric Path Tracing](1863-volumetric-path-tracing.md) which typically relies on voxel grids or implicit fields requiring numerical integration, VPPT provides closed-form analytical solutions for transmittance and free-flight sampling, eliminating Monte Carlo noise for single-scattering events and enabling efficient hardware-accelerated traversal. It maintains 3D volumetric density and integrates physically, preserving reciprocity and avoiding the view-dependent ordering artifacts and popping seen in delta-function approximations like [3D Gaussian Splatting](https://nvlabs.github.io/3dgs/).

### [Unidirectional Differentiable Path Tracing](7780-unidirectional-differentiable-path-tracing.md)
Unidirectional Differentiable Path Tracing is an algorithm to estimate the interior integral of the differential path integral using unidirectional path tracing, computing both the radiometric measurement $L$ and its scene derivative $\dot{L}$. It sets itself apart from standard unidirectional path tracing by jointly estimating the measurement and its gradient with respect to scene parameters $\pi$ using automatic differentiation, while carefully separating PDF terms from the differentiation graph to ensure correctness. It builds on [Unidirectional Path Tracing](2558-unidirectional-path-tracing.md).

### [Path-Space Correlated Rendering](11168-path-space-correlated-rendering.md)
Path-Space Correlated Rendering is a technique that introduces correlation in path space by evaluating two states of the scene (typically the current and a previous state) simultaneously during a single path trace pass. This allows for the computation of gradients or differences between states, enabling the use of control variates for variance reduction in inverse rendering and optimization tasks. It sets itself apart from standard unidirectional path tracing by maintaining dual radiance accumulators and beta weights to track the contribution of both the current and previous scene configurations within the same ray traversal.

### CPU-based Volumetric Path Tracing with MIS
A rendering technique using a CPU-based volumetric path tracer with Multiple Importance Sampling (MIS) to render generated volumetric data across different LoDs [S4]. It uses Woodcock’s Delta Tracking technique for transmittance estimation during the path tracing process [S4].

### Path Tracing Integrator for Material Reconstruction
Use of a path tracing integrator to reconstruct Principled BSDFs, providing an unbiased estimate of the rendering equation and allowing gradients to flow through all physically relevant light transport paths, including indirect bounces and coupled terms [S5, unofficial]. Path tracing is required over compute-efficient integrators like Path Replay Backpropagation because translucent effects and nonlinear interactions with incoming light cannot be captured by the latter [S5, unofficial]. Path tracing requires more samples per pixel to reconstruct textures used to parameterize the material [S5, unofficial].

### Unidirectional Path Tracing with Null-Scattering MIS
A path sampling technique where a path is constructed from the camera, using a random color component to guide distance and albedo-based scattering event type sampling for the entire path [S6]. It combines with Next-Event Estimation (NEE) using Multiple Importance Sampling (MIS) weights derived from exact full path PDFs [S6]. In the dense bunny scene, the method performs better than directional MIS, while in thin media it has a slight performance loss due to the higher cost of ratio tracking, but this is small compared to improved efficiency in dense media [S6]. Unlike directional MIS which only considers the pdf of the shadow ray direction for delta tracking, this method uses exact full path PDFs, allowing it to leverage ratio tracking in NEE for better performance in dense media [S6].

### Unidirectional Path Tracing with NEE and Ratio Tracking
In dense media, this method performs better than directional MIS because it leverages the more efficient ratio tracking technique, whereas directional MIS uses delta tracking along shadow rays [S7]. Ratio tracking is more expensive than delta tracking, causing a slight performance loss in thin media compared to directional MIS [S7].

## Operating characteristics
*   **Complexity:** Time complexity is proportional to the number of bounces and the cost of intersection queries [S2]. Memory complexity is dominated by the scene representation and the recursion stack (or explicit path buffer) [G].
*   **Performance:** Requires significant computational power; an RTX 3090 can run related voxel-based path tracing at 4K averaging around 120fps, but performance dips with scene complexity [S1, unofficial].
*   **Strengths:** Physically accurate global illumination; unbiased estimator; handles complex lighting scenarios including indirect bounces and translucent effects [S5, unofficial][S6].
*   **Limitations:** High variance (noise) requiring many samples per pixel; slow convergence for scenes with complex light transport (e.g., caustics, deep shadows); requires more samples than compute-efficient integrators for material reconstruction [S5, unofficial][S6].
*   **Usage:** Useful for studying visible light and any type of electromagnetic radiation due to its ray-based nature [S2]. Essential for differentiable rendering and material reconstruction where gradients must flow through all light transport paths [S5, unofficial][S6].

## Background
Unidirectional path tracing is the standard algorithmic realization of ray tracing for lighting, tracing sequences of connected rays from the camera to synthesize images by collecting light paths connecting pixels to emitters [S2]. It is widely used in production rendering and research for its physical accuracy [S1, S3, unofficial]. It serves as the foundation for more advanced techniques like differentiable path tracing and volumetric path tracing [S1, S3, unofficial].

## Sources
[S1] jamestkiernan/webgpu-voxel-raymarching, https://github.com/jamestkiernan/webgpu-voxel-raymarching, fetched 2026-10-24 (unofficial)
[S2] Differentiable visual computing for inverse problems and machine learning, https://doi.org/10.1038/s42256-023-00743-0, fetched 2026-10-27
[S3] GitHub - yumcyaWiz/volpt: Volumetric path tracer written in C++, https://github.com/yumcyaWiz/volpt, fetched 2026-10-27 (unofficial)
[S4] Fast Voxelization and Level of Detail for Microgeometry Rendering, https://doi.org/10.48550/arxiv.2604.13191, fetched 2026-10-26
[S5] MRD: Using Physically Based Differentiable Rendering to Probe Vision Models for 3D Scene Understanding, https://doi.org/10.48550/arXiv.2512.12307, fetched 2026-10-29 (unofficial)
[S6] PDF A null-scattering path integral formulation of light transport, https://cs.dartmouth.edu/~wjarosz/publications/miller19null.pdf, fetched 2026-10-29
[S7] PDF A null-scattering path integral formulation of light transport, https://blogs.autodesk.com/media-and-entertainment/wp-content/uploads/sites/162/s2019_null_path_formulation_rev2.pdf, fetched 2026-10-29
