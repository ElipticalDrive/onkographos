# Ray Marching and Discretization

Also called: Ray Marching (Core Algorithm), Ray Marching for Volume Rendering

## Overview
Ray marching is an iterative numerical algorithm used to solve the volume rendering equation by casting rays from the camera, taking discrete steps along the ray, querying volumetric properties, and accumulating contributions via front-to-back alpha compositing [S1][S5, unofficial][S6][S7, unofficial]. It solves the problem of visualizing semi-transparent volumetric data without explicit surface extraction by approximating the continuous light transport integral with discrete samples [S1][S5, unofficial][S6]. Unlike [Ray Traversal Schemes](8341-ray-traversal-and-integration-schemes.md) which focus on efficient voxel traversal logic, ray marching focuses on the integration loop itself, often using fixed or adaptive step sizes [S3][S5, unofficial]. It differs from [Fixed Step Ray Marching](3136-fixed-step-ray-marching.md) in that it can support variable step sizes guided by distance estimators or density, though fixed-step is a common implementation [S3, S6].

## How it works
**Inputs:**
*   Ray origin $\mathbf{o}$ and direction $\mathbf{d}$ (normalized) in world or model space.
*   Volume bounds: $\mathbf{B}_{min}$ and $\mathbf{B}_{max}$ (axis-aligned bounding box).
*   Volume data: 3D texture or procedural function $V(\mathbf{p})$ returning scalar density/value.
*   Transfer function: Maps scalar value to color $\mathbf{c}$ and opacity $\alpha$.
*   Parameters: Step size $\Delta s$ (or number of steps $N$), maximum transmittance threshold for early termination.

**Outputs:**
*   Final pixel color $\mathbf{C}_{final}$ and accumulated opacity $\alpha_{acc}$.

**Data Structures:**
*   **Ray State:** Current position $\mathbf{p}$, accumulated color $\mathbf{C}_{acc}$ (RGB), accumulated opacity $\alpha_{acc}$ (scalar).
*   **Volume Texture:** 3D array or texture with dimensions $W \times H \times D$. Values are typically normalized to $[0, 1]$ or mapped via a colormap.

**Steps:**

1.  **Ray-Box Intersection:** Determine the entry ($t_{start}$) and exit ($t_{end}$) points of the ray with the volume bounding box. If no intersection, return background color [S4, unofficial][S6].
    $$ t_{near} = \max \left( \frac{B_{min} - \mathbf{o}}{\mathbf{d}} \right), \quad t_{far} = \min \left( \frac{B_{max} - \mathbf{o}}{\mathbf{d}} \right) $$
    If $t_{near} > t_{far}$ or $t_{far} < 0$, the ray misses the volume [S4, unofficial].

2.  **Initialization:**
    *   Set $\mathbf{C}_{acc} = \mathbf{0}$, $\alpha_{acc} = 0$.
    *   Calculate step size $\Delta s$. If using fixed steps $N$: $\Delta s = (t_{far} - t_{near}) / N$ [S5, unofficial][S6].
    *   Set current position $\mathbf{p} = \mathbf{o} + t_{near} \mathbf{d}$ [S6].

3.  **Marching Loop:** Iterate while $\alpha_{acc} < \alpha_{threshold}$ (e.g., 0.95 or 1.0) and $\mathbf{p}$ is within bounds [S6][S12, unofficial]:
    a.  **Sample:** Query the volume at $\mathbf{p}$ to get scalar value $v = V(\mathbf{p})$ [S2, unofficial][S6].
    b.  **Transfer Function:** Map $v$ to color $\mathbf{c}$ and base opacity $\alpha_{base}$ using a transfer function or colormap [S2, S4, unofficial][S6].
    c.  **Opacity Correction:** Adjust opacity for the step size to approximate continuous attenuation. A common approximation is $\alpha = 1 - (1 - \alpha_{base})^{\Delta s / \Delta s_{ref}}$ or simply $\alpha = \alpha_{base} \cdot \Delta s$ depending on the physical model (e.g., Beer-Lambert) [S11, S32, S4, unofficial]. For simple alpha blending, $\alpha$ may be directly used if $\alpha_{base}$ is already scaled [S6].
    d.  **Compositing:** Update accumulated color and opacity using front-to-back blending [S1][S13, unofficial]:
        $$ \mathbf{C}_{acc} = \mathbf{C}_{acc} + (1 - \alpha_{acc}) \cdot \alpha \cdot \mathbf{c} $$
        $$ \alpha_{acc} = \alpha_{acc} + (1 - \alpha_{acc}) \cdot \alpha $$
    e.  **Advance:** Move to next sample: $\mathbf{p} = \mathbf{p} + \Delta s \cdot \mathbf{d}$ [S6].

4.  **Final Output:** Composite the accumulated color over a background color $\mathbf{C}_{bg}$ (often black or white):
    $$ \mathbf{C}_{final} = \mathbf{C}_{acc} + (1 - \alpha_{acc}) \cdot \mathbf{C}_{bg} $$

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def ray_march(ray_origin, ray_direction, volume_bounds, volume_sampler, transfer_function, num_steps):
    # 1. Ray-Box Intersection
    t_min = max((volume_bounds.min - ray_origin) / ray_direction)
    t_max = min((volume_bounds.max - ray_origin) / ray_direction)
    
    if t_min > t_max or t_max < 0:
        return vec3(0.0), 0.0  # No intersection

    # 2. Initialization
    step_size = (t_max - t_min) / num_steps
    current_pos = ray_origin + t_min * ray_direction
    acc_color = vec3(0.0)
    acc_alpha = 0.0
    
    # 3. Marching Loop
    for i in range(num_steps):
        # Early termination
        if acc_alpha >= 0.99:
            break
            
        # Sample volume
        value = volume_sampler(current_pos)
        
        # Transfer function
        color, base_alpha = transfer_function(value)
        
        # Opacity correction (simplified linear approximation for small steps)
        # More accurate: alpha = 1.0 - exp(-sigma_t * step_size)
        alpha = base_alpha * step_size 
        
        # Compositing
        transmittance = 1.0 - acc_alpha
        acc_color += transmittance * alpha * color
        acc_alpha += transmittance * alpha
        
        # Advance
        current_pos += step_size * ray_direction

    # 4. Final Output
    return acc_color, acc_alpha
```

## Subtopics

### [Exponential Approximation Method](2079-exponential-approximation-method.md)
The Exponential Approximation Method (EAM) approximates the continuous light attenuation integral using discrete steps and an exponential transmittance model. It differs from standard linear alpha compositing by scaling sampled alpha values by a step-size-dependent correction factor, allowing larger steps while approximating exponential decay. It is typically chosen when performance is prioritized over exact physical accuracy, or when the volume data is sparse enough that early ray termination provides significant speedup.

### [Front-to-Back Volume Rendering](2350-front-to-back-volume-rendering.md)
Front-to-Back Volume Rendering is a direct volume rendering technique that generates images of 3D scalar fields by casting rays from the camera through each pixel, sampling the volume along the ray, and accumulating color and opacity from front to back. It solves the problem of visualizing semi-transparent volumetric data without explicitly extracting geometric surfaces, mapping scalar voxel values to optical properties (color and opacity) via a transfer function. Unlike back-to-front methods, it allows for early ray termination when the accumulated opacity is sufficiently high, ignoring light contributions from behind opaque regions. It differs from [Exponential Approximation Method](2079-exponential-approximation-method.md) by using discrete alpha compositing rather than exponential transmittance approximations, and from [Ray Marching Volume Rendering with Henyey-Greenstein Scattering](2545-ray-marching-volume-rendering-with-henyey-greenstein-scatter.md) by typically neglecting scattering effects in its standard form.

### [Ray Marching Volume Rendering with Henyey-Greenstein Scattering](2545-ray-marching-volume-rendering-with-henyey-greenstein-scatter.md)
This method renders a 3D volume defined by a density texture by accumulating scattered light along a ray, utilizing the Henyey-Greenstein phase function to model anisotropic scattering and shadow rays for self-shadowing. It distinguishes itself from standard front-to-back volume rendering by explicitly incorporating directional light transport through the Henyey-Greenstein phase function, allowing for the visualization of forward or backward scattering effects in participating media.

### [Ray Tracing (Binary Objects)](3002-ray-tracing-binary-objects.md)
A direct volume rendering method for visualizing opaque surfaces contained in binary volumetric data by determining only the first intersection points with a surface. It differs from gray-level volume rendering (e.g., [Front-to-Back Volume Rendering](2350-front-to-back-volume-rendering.md)) by stopping traversal at the first surface hit instead of compositing transmittance and color along the full ray path.

### [Discrete Ray Tracing](3020-discrete-ray-tracing.md)
Discrete Ray Tracing is a volume rendering technique that traverses a voxel grid along a ray using a three-dimensional form of Digital Differential Analysis (DDA). It solves the problem of efficiently sampling scalar data in a regular grid without requiring complex geometric intersection tests or acceleration structures. Unlike [Exponential Approximation Method](2079-exponential-approximation-method.md) which approximates transmittance, or [Fixed Step Ray Marching](3136-fixed-step-ray-marching.md) which uses uniform steps, Discrete Ray Tracing guarantees that the first intersection found is the closest by traversing voxels in the exact order the ray intersects them, limiting intersection tests to Pythagorean complexity rather than cubic.

### [Fixed Step Ray Marching](3136-fixed-step-ray-marching.md)
Fixed Step Ray Marching is a direct volume rendering technique that integrates light transport through participating media (such as fog, smoke, or clouds) by sampling a density field at uniform intervals along a camera ray. It solves the problem of visualizing heterogeneous, translucent volumes without requiring explicit mesh extraction, relying on a fixed discretization of the ray path rather than adaptive stepping or signed distance field (SDF) intersection. Unlike [Ray Tracing (Binary Objects)](3002-ray-tracing-binary-objects.md) which uses SDFs for geometry intersection, this method performs continuous integration of in-scattered light and extinction using the Beer-Lambert law, distinguishing it from [Discrete Ray Tracing](3020-discrete-ray-tracing.md) by its focus on volumetric absorption and emission rather than surface reflection.

### [Volume Ray Casting](3161-volume-ray-casting.md)
Image-order Ray-Casting is a direct volume rendering technique that synthesizes a 2D projection from 3D scalar or vector data by casting primary rays from the viewpoint through each pixel of the image plane into the volumetric dataset. It solves the problem of visualizing semi-transparent, fuzzy, or weakly bounded structures (such as medical tissues or fluid dynamics) without requiring explicit geometric surface extraction. Unlike [Surface Ray Tracing](3002-ray-tracing-binary-objects.md) which terminates at explicit boundaries and spawns secondary rays, or [Splatting](3560-per-primitive-ray-sampling.md) which projects voxels onto the image plane (object-order), Ray-Casting samples the volume continuously along viewing rays and composites optical properties using the Porter-Duff over operator.

### [Per-primitive Ray Sampling](3560-per-primitive-ray-sampling.md)
Per-primitive ray sampling is a rendering strategy where each geometric primitive (specifically ellipsoids) independently updates only the ray segments it intersects, avoiding redundant parameter loads and enabling accurate Transfer Function (TF) mapping by accumulating contributions in 3D space. It sets itself apart from sibling methods like [Fixed Step Ray Marching](3136-fixed-step-ray-marching.md) and [Volume Ray Casting](3161-volume-ray-casting.md) by organizing computation around primitives rather than rays, which eliminates the need for explicit per-tile sorting required in 2D rasterization because contributions accumulate in depth order as each ray segment corresponds to a discrete depth interval.

### [Directional Derivative Diffuse Lighting](4113-directional-derivative-diffuse-lighting.md)
Directional Derivative Diffuse Lighting is an approximation for diffuse lighting in volumetric raymarching that estimates light scattering by comparing density at the current point and an offset point in the light direction, avoiding the need for normal calculation. It sets itself apart from standard surface raymarching normals by requiring only two density samples instead of multiple gradient estimates, making it significantly cheaper for the resource-intensive volumetric loop.

### [Specific Material and Geometry Rendering](4395-specific-material-and-geometry-rendering.md)
Specific Material and Geometry Rendering defines how volumetric data is mapped to visual appearance and how specific projection requirements are handled during ray marching. It solves the problem of decoupling physical density or scattering data from the final pixel color computation, allowing distinct rendering modes and material parameterization to be applied per sample point. It differs from general ray marching by explicitly exposing material interfaces and rendering modes that alter the fragment shader's output logic, rather than relying solely on the integration of the radiative transfer equation.

### [Volumetric Raymarching with SDF Density](4484-volumetric-raymarching-with-sdf-density.md)
Volumetric Raymarching with SDF Density is a rendering technique that adapts sphere tracing principles to volumetric Signed Distance Fields (SDFs), encoding both interior density and exterior distance in a single continuous function. It solves the problem of rendering intricate procedural volumes, such as fractals or noise-based densities, without requiring explicit voxelization or precomputed grids. Unlike standard raymarching which stops at the first surface intersection, this method marches through the object interior using a constant step size, accumulating color and alpha via front-to-back blending based on the SDF's positive interior values. It sets itself apart from [Fixed Step Ray Marching](3136-fixed-step-ray-marching.md) by interpreting the distance field as density rather than geometric proximity, and from [Front-to-Back Volume Rendering](2350-front-to-back-volume-rendering.md) by evaluating the density function procedurally on-the-fly rather than sampling a discrete grid.

### [DOSRender](5119-dosrender.md)
DOSRender is a depth-ordered, slice-based volume rendering technique that accumulates color and occlusion along a ray while simultaneously tracking semantic instance and group IDs for segmentation. It distinguishes itself from standard front-to-back ray casting by integrating precomputed occlusion samples and hierarchical group membership buffers directly into the accumulation pass, allowing for efficient occlusion-aware rendering and semantic data extraction in a single pass.

### [Light Transmittance Marching](5400-light-transmittance-marching.md)
Light Transmittance Marching is a technique to compute physically accurate lighting in a volume by casting a secondary raymarch from each sample point on the primary view ray towards the light source to calculate absorbed light (transmittance). It solves the problem of volumetric occlusion, where light passing through the volume before reaching a sample point is attenuated by the medium itself. Unlike [Front-to-Back Volume Rendering](2350-front-to-back-volume-rendering.md), which typically assumes uniform or pre-computed lighting, this method dynamically computes the light energy arriving at each sample point by accounting for the density along the path to the light.

### Object-Order Ray-Casting with Randomized Sampling
A ray-casting algorithm on rectilinear grids using object-order processing, randomized high sampling rate, and trilinear interpolation [S8]. It initializes a sampling rate variable $\nabla s$ to 0.5, allowing between two and four samplings along a ray within a cell [S8]. It uses fixed pixel subdivisions of $16^2$ (256 subdivisions) and precomputed rays [S8]. A drawback is the lack of perspective projection, which is unsuitable for stereo viewing or VR, though parallel projection is often preferred in scientific visualization [S8].

### Low Albedo Discrete VRE (Alpha Blending Form)
A discrete VRE form derived by approximating transmittance using the first two terms of its Taylor series expansion and assuming low albedo ($\sigma_t \approx \sigma_a, \sigma_s \approx 0$) [S9]. The opacity is approximated as $\alpha_i \approx \sigma_a(\mathbf{p}_i, \omega)\Delta s$ [S9]. The final color is computed as $C = \sum_{i=0}^{N-1} [ C_i \alpha_i \prod_{j=0}^{i-1} t_j ]$, where $C_i$ refers to emission $L_e$ [S9]. This differs from standard discrete VRE where $C_i$ refers to the source term $L_i \Delta s$ [S9].

### Beam Pre-pass
A coarse pre-pass that computes first-hit distances at low resolution to fast-forward the main raymarch pass [S10, unofficial]. It runs at 1/8 resolution, launching one ray per $8 \times 8$ pixel block [S10, unofficial]. Each ray walks the world at tile granularity and writes the first-hit distance to an `r32float` texture [S10, unofficial]. The main pass fast-forwards each ray to that depth minus a 16-voxel safety margin [S10, unofficial].

### Hybrid Rendering
A rendering technique combining semi-transparent volume representation with surface-based mesh representations, setting $t_{max}$ to mesh depth for intersecting rays [S11]. For rays intersecting the mesh, ray marching stops at $t_{max}$ (mesh depth) and fills remaining color throughput with the color of the mesh at the intersection point [S11].

### ROVER Volume Rendering Pipeline
A distributed ray casting volume rendering method that initializes rays with a fixed direction, computes a uniform sample distance from domain bounds, and delegates the marching loop to a VTK-m proxy tracer [S12, unofficial]. Ray direction components are initialized to $(1.0, 1.0, 1.0)$ and the fourth channel (weight/alpha) to 0.0 [S12, unofficial]. The sample distance is calculated as $sample\_distance = \text{magnitude}(\text{domain\_extent}) / \text{num\_samples}$ [S12, unofficial]. A pitfall is that calling `partial_trace` without a valid dataset or primary field causes a runtime error [S12, unofficial].

### VCP Ray-Casting Algorithm
For each ray, this algorithm simultaneously computes the first and last intersection points with the object's bounding box [S13, unofficial]. It marches rays through the VCP grid using varying step-sizes. If the ray hits an empty block, it reads the scalar distance value; if it hits a non-empty block, it uses the length of the interpolated VCP vector [S13, unofficial]. If the current distance is less than $\epsilon$, it assumes a ray-surface intersection and interpolates to find the exact point [S13, unofficial]. It evaluates a local lighting model using the normalized VCP vector as the surface normal [S13, unofficial].

### Parallel Analytic Marching
A parallelization strategy where multiple initial points are solved simultaneously to march towards all unsolved analytic cells in the active set [S14]. It initializes multiple points $\mathbf{x}$ in $\mathbb{Z}$ in parallel by solving Eq 17, instead of a single point [S14]. This provides an efficiency improvement of order 10 on Nvidia GPUs compared to sequential execution [S14].

## Operating characteristics
*   **Time Complexity:** $O(N \cdot P)$, where $N$ is the number of samples per ray and $P$ is the number of pixels [S4, unofficial][S6]. Performance scales linearly with the number of steps [S4, unofficial].
*   **Memory Complexity:** $O(W \cdot H \cdot D)$ for the volume texture storage [S6].
*   **Performance:** Fixed-step ray marching offers predictable performance and is well-suited for GPU implementation in fragment shaders [S4, unofficial][S6]. Early ray termination can significantly speed up rendering for sparse volumes [S1, S6].
*   **Strengths:** Simple to implement, supports real-time performance on GPUs, handles semi-transparent data naturally [S1][S4, unofficial][S6].
*   **Limitations:** Fixed step sizes can lead to aliasing or missing details if steps are too large; adaptive stepping adds complexity [S3][S4, unofficial]. Early termination may miss subtle details in transparent regions [S6].
*   **When to use:** Real-time visualization of volumetric data, medical imaging, procedural volumes [S1][S4, unofficial][S6].
*   **When not to use:** When high physical accuracy with complex scattering is required (consider Monte Carlo methods) [S12, unofficial].

## Background
Ray marching was introduced as an efficient technique for solving the volume rendering equation with real-time performance, particularly in the context of GPU-based rendering [S1][S2, unofficial]. It has been widely adopted in various frameworks, including Jrender for differentiable rendering [S1], Three.js for web-based volume visualization [S4, unofficial], and Qt Quick 3D for desktop applications [S6]. The method is foundational for rendering Neural Radiance Fields (NeRF) and Signed Distance Functions (SDFs) [S5, unofficial].

## Sources
[S1] Jrender: An efficient differentiable rendering library based on Jittor, https://doi.org/10.1016/j.gmod.2023.101202, fetched 2026-10-27
[S2] Dy111111/volumeRenderingGPU, https://github.com/Dy111111/volumeRenderingGPU, fetched 2026-10-26 (unofficial)
[S3] Volume ray casting — Grokipedia, https://grokipedia.com/page/Volume_ray_casting, fetched 2026-10-26
[S4] here, https://github.com/Donitzo/three.js-volume-renderer, fetched 2026-10-28 (unofficial)
[S5] Volume Rendering Integral: Definition & AI/ML Use, https://inferensys.com/glossary/neural-radiance-fields-and-spatial-computing/neural-scene-representations/volume-rendering-integral, fetched 2026-10-28 (unofficial)
[S6] Qt Quick 3D - 体积渲染示例, https://doc.qt.io/qt-6/zh/qtquick3d-volumeraycaster-example.html, fetched 2026-10-30
[S7] Raymarching with Fennel and LÖVE · Andrey Listopadov, https://andreyor.st/posts/2020-10-15-raymarching-with-fennel-and-love/, fetched 2026-10-30 (unofficial)
[S8] A new object-order ray-casting algorithm, https://doi.org/10.5555/602099.602130, fetched 2026-10-26
[S9] 14 Rendering Volume and Subsurface Scattering - Foundations of Visual ..., https://learnvisualcomputing.github.io/rendering-rte.html, fetched 2026-10-27
[S10] mstampfli/voxelG, https://github.com/mstampfli/voxelG, fetched 2026-10-24 (unofficial)
[S11] Neural Volumes: Learning Dynamic Renderable Volumes from Images, https://arxiv.org/abs/1906.07751, fetched 2026-10-26
[S12] llnl/rover, https://github.com/llnl/rover, fetched 2026-10-26 (unofficial)
[S13] Vector-to-Closest-Point Octree for Surface Ray-Casting, https://doi.org/10.2312/vmv.20151259, fetched 2026-10-26 (unofficial)
[S14] Learning and Meshing From Deep Implicit Surface Networks Using an Efficient Implementation of Analytic Marching, https://doi.org/10.1109/tpami.2021.3135007, fetched 2026-10-26
