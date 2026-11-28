# Metaball Field Evaluation

Also called: Dynamic Metaball Volume Update Pipeline, GPU Field Generation, GPU volume generation, Implicit Metaball Field, Metaball Density Function, Metaball Field Computation, Metaball density field update

## Overview
Metaball Field Evaluation generates and updates a 3D scalar density volume representing dynamic metaballs (spheres) using a CPU-GPU pipeline, typically via compute or fragment shaders [S1, S3, unofficial]. It solves the problem of converting discrete, moving atomic positions into a continuous volumetric representation suitable for surface extraction or raymarching [S1, unofficial]. Unlike [Implicit Field Composition](2652-implicit-field-composition.md), which composes general implicit primitives, this method specifically sums inverse-distance or Gaussian contributions from moving spheres to create organic, merging blob shapes [S1, S3, unofficial].

## How it works
**Inputs and Outputs:**
*   **Inputs:** A set of $N$ metaballs, each defined by position $\vec{p}_i$, radius $r_i$, and strength $s_i$; a 3D grid domain of resolution $W \times H \times D$; and a time parameter $t$ [S1, S2, unofficial].
*   **Outputs:** A 3D texture (volume) where each voxel stores the summed scalar field value (density) and optionally color/normal data [S1, S3, unofficial].

**Data Structures:**
*   **Ball Buffer (Constant Buffer):** Stores ball parameters. In DirectX 11 implementations, this is a structured buffer or constant buffer `CB_Balls` containing:
    *   `fBalls[150]`: Array of `XMFLOAT4` (x,y,z = position, w = power/radius squared) [S1, unofficial].
    *   `fBallsCol[150]`: Array of `XMFLOAT4` (color) [S1, unofficial].
    *   `iNumBalls`: Integer count of active balls [S1, unofficial].
    *   Max balls supported is typically 150 [S1, unofficial].
*   **Volume Texture:** `ID3D11Texture3D` with format `DXGI_FORMAT_R32G32B32A32_FLOAT` [S1, unofficial].
    *   Dimensions: $(W, H, D)$, often derived from half-resolution bounds [S1, unofficial].
    *   Layout: XYZ channels store color/intensity, W channel stores density [S1, unofficial].
*   **Flag Volume (Optional):** `ID3D11Texture3D` with format `DXGI_FORMAT_R8_SINT` [S1, unofficial].
    *   Dimensions: Downsampled by `VOXEL_BRICK_RATIO` [S1, unofficial].
    *   Purpose: Marks empty space for skipping during raymarching [S1, unofficial].

**Steps:**
1.  **Initialization (CPU):**
    *   Create volume textures with specified resolutions [S1, unofficial].
    *   Initialize ball parameters:
        *   Radius $r$: Randomized, e.g., $r = (0.6 \cdot \text{rand}() + 0.7) \cdot \text{halfReso} \cdot \text{voxelSize} \cdot 0.1$ [S1, unofficial].
        *   Power $w = r^2$ [S1, unofficial].
        *   Orbit Radius: Randomized within bounds, clamped so $r + \text{orbitRadius} < 0.45 \cdot \text{fullReso}$ [S1, unofficial].
        *   Speed: Randomized, clamped to $\pm 1.0$, scaled by $1/w$ [S1, unofficial].
        *   Phase: Random angle $[0, 2\pi)$ [S1, unofficial].
    *   Upload initial ball data to GPU constant buffers [S1, unofficial].

2.  **Dynamic Update (CPU):**
    *   For each frame, update ball positions based on time $t$:
        *   $x_i(t) = r_i \cos(t \cdot \text{speed}_i + \text{phase}_i)$
        *   $y_i(t) = r_i \sin(t \cdot \text{speed}_i + \text{phase}_i)$
        *   $z_i(t) = 0.3 \cdot r_i \sin(2 \cdot t \cdot \text{speed}_i + \text{phase}_i)$ [S1, unofficial].
    *   Update the `CB_Balls` buffer on the GPU [S1, unofficial].

3.  **Field Evaluation (GPU):**
    *   Dispatch a compute shader or fragment shader to evaluate the field for each voxel [S1, S3, unofficial].
    *   **Voxel Coordinate Mapping:**
        *   Map fragment/pixel coordinates to 3D grid indices $(x, y, z)$ [S3, unofficial].
        *   Normalize to $[0, 1]$: $fx = x/(W-1), fy = y/(H-1), fz = z/(D-1)$ [S3, unofficial].
    *   **Field Calculation:**
        *   Initialize field value $V = 0.0$ [S3, unofficial].
        *   For each ball $i$ (up to $N$):
            *   Calculate squared distance: $d_i^2 = (fx - p_{ix})^2 + (fy - p_{iy})^2 + (fz - p_{iz})^2$ [S2, S3, unofficial].
            *   Calculate contribution: $c_i = \frac{s_i}{d_i^2 + \epsilon}$ [S2, S3, unofficial].
            *   Apply threshold (optional): If $c_i - \text{threshold} > 0$, add to $V$ [S3, unofficial].
        *   Add background/floor contributions if defined (e.g., floor plane at $y=0$) [S3, unofficial].
    *   **Write Output:** Store $V$ (and color) in the volume texture [S1, S3, unofficial].

**Formulas:**
The density $D$ at point $\vec{x}$ is the sum of contributions from $N$ metaballs:
$$ D(\vec{x}) = \sum_{i=1}^{N} \frac{s_i}{\|\vec{x} - \vec{p}_i\|^2 + \epsilon} $$
where:
*   $s_i$ is the strength of ball $i$ [S2, S3, unofficial].
*   $\vec{p}_i$ is the position of ball $i$ [S2, S3, unofficial].
*   $\epsilon$ is a small constant to prevent division by zero (e.g., $0.001$ or $0.000001$) [S2, S3, unofficial].
*   $\|\cdot\|$ denotes Euclidean distance [S2, unofficial].

Some implementations subtract a threshold $T$ before accumulation:
$$ V(\vec{x}) = \sum_{i=1}^{N} \max\left(0, \frac{s_i}{\|\vec{x} - \vec{p}_i\|^2 + \epsilon} - T\right) $$
where $T$ is typically $12.0$ [S3, unofficial].

**Parameters:**
*   $\epsilon$: Epsilon for stability. Values: $0.001$ [S2, unofficial], $0.000001$ [S3, unofficial].
*   Threshold $T$: Isosurface cutoff. Value: $12.0$ [S3, unofficial].
*   Grid Size: e.g., $384 \times 384 \times 384$ [S1, unofficial].
*   Max Balls: 150 [S1, unofficial].

**Edge Cases and Pitfalls:**
*   **Precision:** Avoid converting large integer voxel indices to floats in shaders if latency is a concern; store resolutions as floats in constant buffers [S1, unofficial].
*   **Boundary Conditions:** Discard fragments outside the valid grid bounds $[0, W-1] \times [0, H-1] \times [0, D-1]$ [S3, unofficial].
*   **Clamping:** Ensure ball orbits do not exceed the volume bounds to prevent artifacts [S1, unofficial].

## Subtopics

### [Radial Gaussian Density Field Generation](2081-radial-gaussian-density-field-generation.md)
Generates a 3D scalar density field by summing normalized radial Gaussian functions centered at predefined atomic positions, then scaling and clamping the result to a $[0,1]$ range. It solves the problem of converting discrete atomic coordinates into a continuous volumetric representation suitable for surface extraction. Unlike [Metaball Field Evaluation](1017-metaball-field-evaluation.md), which evaluates a dynamic scalar field composed of moving metaballs, this method uses static, smooth Gaussian falloffs centered on fixed points.

### [Blob Field Generation](2487-blob-field-generation.md)
Blob Field Generation is a procedural method for constructing a 3D scalar field by summing inverse-squared Euclidean distance contributions from a set of source points (metaballs). It solves the problem of defining smooth, organic implicit surfaces for isosurface extraction algorithms like Marching Cubes. It is distinguished from sibling methods by its specific use of the inverse square falloff ($1/r^2$) rather than Gaussian decay or fuzzy hyper-parameter blending, resulting in sharper, more defined "blob" interactions.

### [Fuzzy Metaballs Hyper-parameters](8382-fuzzy-metaballs-hyper-parameters.md)
This family defines the five scalar hyper-parameters used to tune the algebraic intersection and alpha-mask generation stages of the Fuzzy Metaballs rendering method. It solves the problem of converting raw implicit field evaluations into differentiable depth and alpha masks suitable for gradient-based pose optimization. What sets this apart from sibling methods like [Radial Gaussian Density Field Generation](2081-radial-gaussian-density-field-generation.md) or [Blob Field Generation](2487-blob-field-generation.md) is that these parameters are not intrinsic properties of the volume geometry itself, but rather optimization targets for the *rendering pipeline's* differentiability and silhouette accuracy, specifically tuned for algebraic surface approximation.

### Welland Weight Matrix Generation
Generates a diagonal weight matrix for point cloud processing based on Welland functions [S4, unofficial].
*   **Step:** Initialize diagonal matrix $W$ of size $P \times P$ with zeros. For each point $P(i)$, calculate distance $r = \|P(i) - X\|$. Set $W(i,i) = w$.
*   **Formula:** $w = \left(\left(\frac{4r}{h} + 1\right)\left(1 - \frac{r}{h}\right)\right)^4$ [S4, unofficial].
*   **Parameter:** $h$ is the height of the Welland function, used as a scaling factor for radius $r$ [S4, unofficial].

### LerpedSpheres
Generates a 3D density field by blending signed distances of multiple spheres using a smooth minimum function, optionally perturbed by noise [S5, unofficial].
*   **Step:** Calculate bounding box. Initialize density field. Iterate voxels. Compute signed distance for each sphere. If `blend_factor < 0.01`, use simple minimum. Otherwise, iteratively blend distances using `SmoothUnion(min_dist, signed_dist, blend_factor)` [S5, unofficial].
*   **Parameter:** `blend_factor` controls smoothness; `noise_amplitude` controls perturbation [S5, unofficial].

## Operating characteristics
*   **Time Complexity:** $O(V \cdot N)$, where $V$ is the number of voxels and $N$ is the number of metaballs [S1, S3, unofficial]. Each voxel evaluates contributions from all balls.
*   **Memory Complexity:** $O(V)$ for the volume texture, plus $O(N)$ for ball parameters [S1, unofficial].
*   **Performance:** Highly parallelizable on GPU via compute shaders [S1, unofficial]. Performance scales with grid resolution and ball count. Using a flag volume for empty space skipping can reduce raymarching costs but adds overhead to field generation [S1, unofficial].
*   **Strengths:** Real-time dynamic updates; organic merging shapes; simple implementation [S1, S3, unofficial].
*   **Limitations:** $O(N)$ per voxel is expensive for large $N$; fixed grid resolution limits detail; inverse-square falloff can cause high values near centers requiring careful thresholding [S2, S3, unofficial].

## Background
Originated in the context of GPU-accelerated volume rendering and implicit surface generation [S1, S3, unofficial]. Early implementations used CPU updates with GPU texture uploads [S1, unofficial]. Modern approaches use compute shaders for direct GPU-side field generation [S1, S3, unofficial]. The method is widely used in real-time visualization of fluid-like dynamics and organic modeling [S1, S2, unofficial].

## Sources
[S1] pengliu916/DynamicVolumeRayCasting, https://github.com/pengliu916/DynamicVolumeRayCasting, fetched 2026-10-26 (unofficial)
[S2] nff747/marching-cubes-wgsl, https://github.com/nff747/marching-cubes-wgsl, fetched 2026-10-27 (unofficial)
[S3] 3merillon/WebGL2-GPU-Marching-Cubes, https://github.com/3merillon/WebGL2-GPU-Marching-Cubes, fetched 2026-10-28 (unofficial)
[S4] mwrazam/marchingtets-reconstruction, https://github.com/mwrazam/marchingtets-reconstruction, fetched 2026-10-27 (unofficial)
[S5] lijiemark/floating-islands-generator, https://github.com/lijiemark/floating-islands-generator, fetched 2026-10-27 (unofficial)
