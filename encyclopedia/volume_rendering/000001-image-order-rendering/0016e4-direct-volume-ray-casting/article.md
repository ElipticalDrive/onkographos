# Direct volume ray casting


## Overview
Direct volume ray casting is an image-order rendering technique that casts a ray through a 3D volume for every pixel, sampling scalar values, mapping them to RGBA via a transfer function, and compositing samples front-to-back using alpha blending to produce the final image [S2, S3, unofficial]. It differs from [Isosurface and first-hit ray casting](20-isosurface-and-first-hit-ray-casting.md) by allowing all voxels to contribute to the final image rather than stopping at the first opaque surface [S5, unofficial], and from [Distance-field ray marching](21-distance-field-ray-marching.md) by using uniform step sizes rather than distance-field-guided steps [G]. It replaces slice-stack methods by providing a solid volume appearance from any angle without edge-on gaps [S15, unofficial].

## Shared principles
The method relies on the volume rendering integral, discretized into a summation over sample points along a ray [S3, S5, unofficial]. The ray $r$ is parameterized by $r(t) = O + tD$, entering the volume at $t_{start}$ and exiting at $t_{end}$ [Volume Raycasting Discretization](831-volume-raycasting-discretization.md). The data model is sampled using $s(t)$ [Volume Raycasting Discretization](831-volume-raycasting-discretization.md).

The compositing is performed front-to-back [S2, S3, S8, S9, S11, S14, S16, S20, S23, S26, unofficial]. Let $C'_i$ be the accumulated color and $\alpha'_i$ be the accumulated opacity after sample $i$, with initial conditions $C'_0 = 0$ and $\alpha'_0 = 0$ [S9, unofficial]. For each sample $i$ along the ray:
1.  Sample the volume at the current ray position to get a scalar value $d_i$ [S2, S5, unofficial].
2.  Apply a transfer function to map the scalar value to color $C_i$ and opacity $\alpha_i$ [S2, S5, unofficial].
3.  Update accumulated color:
    $$ C'_i = C'_{i-1} + (1 - \alpha'_{i-1}) \alpha_i C_i $$
4.  Update accumulated opacity:
    $$ \alpha'_i = \alpha'_{i-1} + (1 - \alpha'_{i-1}) \alpha_i $$
5.  If $\alpha' \ge 1$ or a threshold (typically 0.99) [S2, S18, unofficial], terminate the ray (early ray termination) [S2, S3, S4, unofficial].

Rays are cast from the eyepoint, one for each pixel [S1][S4, unofficial]. The volume is typically represented as a 3D texture [S4, S3, unofficial]. Gradients for shading are calculated using a central difference operator [S6, unofficial]. To avoid normal discontinuities that cause visible rings on smooth surfaces, the gradient is computed using a two-voxel offset rather than a one-voxel offset [S18, unofficial]. Sampling is performed using trilinear interpolation [S1][S5, unofficial]. A fixed random offset is applied to each ray's start position to prevent sample layers from appearing as bands [S18, unofficial]. The final accumulated color is typically corrected with gamma 2.2 [S9, unofficial].

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import random
import math

def vec3(x, y, z):
    return [x, y, z]

def pow_vec3(v, exponent):
    return [v[0]**exponent, v[1]**exponent, v[2]**exponent]

def compute_ray_exit(origin, direction, bounds):
    # Slab intersection for axis-aligned box [G]
    t0 = max((bounds[0][i] - origin[i]) / direction[i] if direction[i] != 0 else float('inf') for i in range(3))
    t1 = min((bounds[1][i] - origin[i]) / direction[i] if direction[i] != 0 else float('inf') for i in range(3))
    return max(t0, t1)

def trilinear_interpolate(volume, pos, dims):
    # Standard trilinear interpolation [G]
    vx = pos[0] * (dims[0] - 1)
    vy = pos[1] * (dims[1] - 1)
    vz = pos[2] * (dims[2] - 1)
    x0, y0, z0 = int(math.floor(vx)), int(math.floor(vy)), int(math.floor(vz))
    x1, y1, z1 = x0 + 1, y0 + 1, z0 + 1
    x0 = max(0, min(x0, dims[0]-1))
    x1 = max(0, min(x1, dims[0]-1))
    y0 = max(0, min(y0, dims[1]-1))
    y1 = max(0, min(y1, dims[1]-1))
    z0 = max(0, min(z0, dims[2]-1))
    z1 = max(0, min(z1, dims[2]-1))
    wx = vx - x0
    wy = vy - y0
    wz = vz - z0
    c00 = volume[x0][y0][z0] * (1-wx) + volume[x1][y0][z0] * wx
    c01 = volume[x0][y0][z1] * (1-wx) + volume[x1][y0][z1] * wx
    c10 = volume[x0][y1][z0] * (1-wx) + volume[x1][y1][z0] * wx
    c11 = volume[x0][y1][z1] * (1-wx) + volume[x1][y1][z1] * wx
    r0 = c00 * (1-wy) + c10 * wy
    r1 = c01 * (1-wy) + c11 * wy
    return r0 * (1-wz) + r1 * wz

def direct_volume_raycast(volume_data, camera, transfer_function, volume_bounds, volume_dims, step_size=0.001, threshold=0.99):
    """
    Renders a volume using fixed-step raycasting with front-to-back compositing.
    """
    h, w = camera.screen_height, camera.screen_width
    image = [[vec3(0.0, 0.0, 0.0) for _ in range(w)] for _ in range(h)]
    
    for y in range(h):
        for x in range(w):
            ray_origin, ray_direction = camera.cast_ray(x, y)
            accumulated_color = vec3(0.0, 0.0, 0.0)
            accumulated_opacity = 0.0
            
            # Ray initialization: fixed random offset to prevent banding [S18, unofficial]
            t = random.uniform(0.0, step_size)
            
            # Compute ray exit distance from volume bounds [G]
            t_end = compute_ray_exit(ray_origin, ray_direction, volume_bounds)
            
            while accumulated_opacity < threshold and t < t_end:
                # Compute current position in texture space [0,1]
                pos = [(ray_origin[i] + t * ray_direction[i]) for i in range(3)]
                # Sample volume at current position using trilinear interpolation [S1][S5, unofficial]
                scalar_value = trilinear_interpolate(volume_data, pos, volume_dims)
                
                # Apply transfer function to get color and opacity [S2, S5, unofficial]
                color, opacity = transfer_function(scalar_value)
                
                # Front-to-back compositing [S2, S5, unofficial]
                accumulated_color[0] += (1.0 - accumulated_opacity) * opacity * color[0]
                accumulated_color[1] += (1.0 - accumulated_opacity) * opacity * color[1]
                accumulated_color[2] += (1.0 - accumulated_opacity) * opacity * color[2]
                accumulated_opacity += (1.0 - accumulated_opacity) * opacity
                
                t += step_size
                
            # Gamma correction [S9, unofficial]
            image[y][x] = pow_vec3(accumulated_color, 1.0/2.2)
            
    return image
```

## Subtopics

### [Distance Color Blending](791-distance-color-blending.md)
Enhances depth perception by dimming and shifting colors towards a background as they recede [791-distance-color-blending.md].

### [Interpolation Schemes for Volume Rendering](817-interpolation-schemes-for-volume-rendering.md)
Handles reconstruction of scalar values, addressing artifacts from nearest neighbor interpolation.

### [Volume Raycasting Discretization](831-volume-raycasting-discretization.md)
Defines the discretization of the volume rendering integral and ray parameterization context.

### [Maximum Intensity Projection](902-maximum-intensity-projection.md)
Finds the maximum voxel value along the ray, ignoring intermediate values.

### [Alpha Compositing Ray Casting](922-alpha-compositing-ray-casting.md)
Projects a semi-transparent 3D scalar field onto a 2D plane by integrating optical properties along viewing rays.

### [Stochastic Jittering](924-stochastic-jittering.md)
Reduces banding artifacts by adding stochastic jittering controlled by a noise texture.

### [Hierarchical Sampling (Inverse CDF Sampling)](973-hierarchical-sampling-inverse-cdf-sampling.md)
Samples points along a ray from a discrete probability distribution using inverse CDF interpolation.

### [GPU-based Ray-casting for Unstructured Grids](1118-gpu-based-ray-casting-for-unstructured-grids.md)
Guarantees accurate rendering results for non-convex meshes on GPU.

### [Direct Volume Rendering](1184-direct-volume-rendering.md)
Considers volume data as a transparent gel, simulating light travelling through it without mapping.

### [Multi-Sample Anti-Aliasing via Ray Distribution](1219-multi-sample-anti-aliasing-via-ray-distribution.md)
Casts multiple rays per pixel distributed within the pixel area to reduce jagged edges.

### [Uniform Ray Sampling](1261-uniform-ray-sampling.md)
Generates points uniformly distributed between minimum and maximum depth along each ray.

### [Opacity Correction for Slice Count Invariance](1277-opacity-correction-for-slice-count-invariance.md)
Scales per-slice opacity to maintain constant image intensity regardless of slice count.

### [Back-face rasterized volume rendering](1296-back-face-rasterized-volume-rendering.md)
Rasterizes back faces of the bounding box to trigger fragment shaders, avoiding processing pixels outside the volume.

### [Classical Ray Tracing Integration](1334-classical-ray-tracing-integration.md)
Computes pixel color by combining background transmission and uniform in-scattering, applying Beer's law.

### [Procedural Volume Rendering with Noise Perturbation](1370-procedural-volume-rendering-with-noise-perturbation.md)
Enhances coarse macrostructure at render time by perturbing texture coordinates and optical properties with noise.

### [Rendering Pipelines and Passes](1411-rendering-pipelines-and-passes.md)
Manages the sequence of rendering operations for volume data.

### [2D Visualization from 3D Grid (Slow)](1507-2d-visualization-from-3d-grid-slow.md)
Visualizes 3D grid data into 2D views.

### [Trilinear Interpolation](1531-trilinear-interpolation.md)
Renders scalar field data on a 3D rectilinear grid using front-to-back alpha compositing.

### [Single Scattering Volume Rendering with Ray Marching](1532-single-scattering-volume-rendering-with-ray-marching.md)
Simulates light interaction with particles assuming light scatters only once before reaching the camera.

### [Neural volume rendering](1576-neural-volume-rendering.md)
Uses neural networks to represent and render volumetric data.

### [Composition via Convolution (Power Form)](1591-composition-via-convolution-power-form.md)
Transforms a trivariate polynomial into a univariate polynomial using convolution operators.

### [Tile-based Volume Ray Marching](1678-tile-based-volume-ray-marching.md)
Stores 3D volume data in a 1D texture array organized into tiles for ray marching.

### [Ray Marching Volume Rendering](1735-ray-marching-volume-rendering.md)
Real-time GPU method using Taichi to perform ray marching through a 3D scalar field.

### Piecewise Density Approximation for Reparameterized Sampling
An approximation of the opacity function $F_r(t)$ and its inverse by approximating the density field $\sigma_r(t)$ as piecewise constant or piecewise linear on a grid, allowing analytical computation of the integral $\int \sigma_r(s) ds$ [S29, unofficial]. For piecewise constant approximation, pick a point $\hat{t}_i$ within each bin $[t_i, t_{i+1}]$ and approximate density as $\sigma_r(\hat{t}_i)$ inside the bin [S29, unofficial]. For piecewise linear approximation, compute $\sigma_r$ at grid points and interpolate values between grid points [S29, unofficial]. Solve $y_f u = 1 - \exp(-I(t))$ for $t$, where $I(t)$ is the chosen integral approximation ($I_0$ or $I_1$), to find the sample location $t$ corresponding to uniform sample $u$ [S29, unofficial]. The integral approximation $I_0(t)$ for piecewise constant density is: $I_0(t) = \sum_{j=1}^i \sigma_r(\hat{t}_j)(t_j - t_{j-1}) + \sigma_r(\hat{t}_i)(t - t_i)$, where $t \in [t_i, t_{i+1}]$ [S29, unofficial]. The integral approximation $I_1(t)$ for piecewise linear density is a sum of trapezoidal areas [S29, unofficial]. The grid is a set of points $t_n = t_0 < t_1 < \dots < t_m = t_f$ defining the bins for the piecewise approximation [S29, unofficial].

### Projected Slabs Approximation
An algorithm that approximates perspective volume rendering using parallel projected slabs to enable acceleration on hardware designed for parallel projection, with analytical error bounds [S30, unofficial]. It approximates perspective projection using parallel projected slabs, performs an analytical study of maximum and average error, and improves the basic algorithm based on error analysis to bound global maximum error while increasing frame rate [S30, unofficial].

### Forward Projection (aTx)
Inside the loop, for each intersected voxel (img_i, img_j), add length * image_value to the sum, where image_value is accessed at index batch_index * channels * height * width + channel_index * height * width + img_i * width + img_j [S31, unofficial].

### Commutative Blend Direct Volume Rendering
Rendering technique for unsorted cell projection using optical models that lead to a commutative blend function, allowing compositing in arbitrary order [S32]. It applies arbitrary transfer functions via texture lookup, which only affect the generation of the texture map [S32]. Models allowing arbitrary order compositing include maximum intensity projection and restrictions of the full volume density model to either the source term or the absorption term [S32].

## Background
The term "direct volume rendering" was coined by Marc Levoy in his 1988 paper "Display of Surfaces from Volume Data" [S6, unofficial]. Levoy defined the volume rendering pipeline, which remains an influential framework [S6, unofficial]. The technique was originally designed to display weak or fuzzy surfaces with smooth silhouettes and few aliasing artifacts [S6, unofficial].

## Sources
[S1] Hierarchical and parallelizable direct volume rendering for irregular and multiple grids, https://doi.org/10.1109/visual.1996.567606, fetched 2026-10-24
[S2] Interactive volume ray tracing, https://doi.org/10.22028/d291-25931, fetched 2026-10-26 (unofficial)
[S3] Advanced illumination and view-selection techniques for volume rendering and its application to medical imaging, http://hdl.handle.net/10803/371154, fetched 2026-10-27 (unofficial)
[S4] A system for real-time rendering of compressed time-varying volume data, https://doi.org/10.7939/r3041m, fetched 2026-10-26 (unofficial)
[S5] Shear-warp factorization volume rendering visualization system, https://archive.org/details/0162018299360, fetched 2026-10-26 (unofficial)
[S6] Volume Visualisation Via Variable-Detail Non-Photorealistic Illustration, https://uwspace.uwaterloo.ca/bitstream/10012/1104/1/jlmckinl2002.pdf, fetched 2026-10-26 (unofficial)
[S7] DakaiZhou/Volume-Rendering, https://github.com/DakaiZhou/Volume-Rendering, fetched 2026-10-26 (unofficial)
[S8] rhnfzl/raycasting-volume-rendering, https://github.com/rhnfzl/raycasting-volume-rendering, fetched 2026-10-26 (unofficial)
[S9] upupming/volume-rendering, https://github.com/upupming/volume-rendering, fetched 2026-10-26 (unofficial)
[S10] khizarsiddiqui/Volume-Rendering, https://github.com/khizarsiddiqui/Volume-Rendering, fetched 2026-10-26 (unofficial)
[S11] Twinklebear/webgl-marching-cubes, https://github.com/Twinklebear/webgl-marching-cubes, fetched 2026-10-26 (unofficial)
[S12] l0exy1h/VolumetricPathTracing-public, https://github.com/l0exy1h/VolumetricPathTracing-public, fetched 2026-10-26 (unofficial)
[S13] Yonghao-Lee/differentiable-rendering, https://github.com/Yonghao-Lee/differentiable-rendering, fetched 2026-10-26 (unofficial)
[S14] viljanvr/VisualNifty, https://github.com/viljanvr/VisualNifty, fetched 2026-10-26 (unofficial)
[S15] vulekhanh/VolumeRendering, https://github.com/vulekhanh/VolumeRendering, fetched 2026-10-26 (unofficial)
[S16] GitHub - LDeakin/VkVolume: Vulkan-based volume renderer with efficient ..., https://github.com/LDeakin/VkVolume, fetched 2026-10-26 (unofficial)
[S17] A Parallel, Feed-Forward Volume Rendering Algorithm, https://www.cs.unc.edu/techreports/91-029.pdf, fetched 2026-10-27 (unofficial)
[S18] 3D: ray-cast volume rendering instead of a stack of slices, https://github.com/jamescarruthers/quick_dicom/pull/4, fetched 2026-10-27 (unofficial)
[S19] urielfederez/Volume-Data-Visualization, https://github.com/urielfederez/Volume-Data-Visualization, fetched 2026-10-27 (unofficial)
[S20] kapadia/volumetric.js, https://github.com/kapadia/volumetric.js, fetched 2026-10-27 (unofficial)
[S21] PDF Direct Volume Rendering, https://www.sci.utah.edu/~stevec/classes/cs5630/dvr.pdf, fetched 2026-10-27 (book)
[S22] Volume and Isosurface Rendering with GPU‐Accelerated Cell Projection, https://www.researchgate.net/publication/227523087_Volume_and_Isosurface_Rendering_with_GPU-Accelerated_Cell_Projection, fetched 2026-10-27 (unofficial)
[S23] franjaviersans/VRCUDA, https://github.com/franjaviersans/VRCUDA, fetched 2026-10-27 (unofficial)
[S24] Procedural Volumetric Clouds · Jan Wedekind, https://www.wedesoft.de/software/2023/05/03/volumetric-clouds/, fetched 2026-10-28 (unofficial)
[S25] (PDF) GPU-Based Volume Rendering for Medical Imagery - ResearchGate, https://www.researchgate.net/publication/228912978_GPU-Based_Volume_Rendering_for_Medical_Imagery, fetched 2026-10-28 (unofficial)
[S26] (PDF) Differentiable Direct Volume Rendering - ResearchGate, https://www.researchgate.net/publication/353510271_Differentiable_Direct_Volume_Rendering, fetched 2026-10-30 (unofficial)
[S27] dongeronimo/webgpu_volume_renderer_demo, https://github.com/dongeronimo/webgpu_volume_renderer_demo, fetched 2026-10-29 (unofficial)
[S28] Texture Mapping Based Volume Rendering, https://www.ciroduran.com/files/Tesis.pdf, fetched 2026-10-29 (unofficial)
[S29] Differentiable Rendering with Reparameterized Volume Sampling, https://doi.org/10.48550/arxiv.2302.10970, fetched 2026-10-26 (unofficial)
[S30] Academia | (PDF) FAST VOLUME RENDERING USING A SHEAR-WARP FACTORIZATION ..., https://www.academia.edu/6981092/FAST_VOLUME_RENDERING_USING_A_SHEAR_WARP_FACTORIZATION_OF_THE_VIEWING_TRANSFORMATION, fetched 2026-10-27 (unofficial)
[S31] ApaarRaina/DiffTomo, https://github.com/ApaarRaina/DiffTomo, fetched 2026-10-27 (unofficial)
[S32] Hardware-based view-independent cell projection, https://www.semanticscholar.org/paper/Hardware-based_view-independent_cell_projection-Weiler-Kraus/36615d19709bce724437ec3257e43a1916f7de26, fetched 2026-10-29
