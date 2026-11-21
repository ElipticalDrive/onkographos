# Ray Casting and Compositing Loop

Also called: Basic Ray Casting, Ray Casting Volume Rendering, Single pass GPU Ray casting, Standard Ray Casting Loop

## Overview
Ray Casting and Compositing Loop is a single-pass GPU method for direct volume rendering that renders scalar data using front-to-back compositing [S1, unofficial]. It samples scalar values along rays and maps them directly to color and alpha without explicit surface extraction [S1, unofficial]. Unlike IsoSurface Ray Marching, this method does not detect zero crossings or compute normals [S1, unofficial].

## How it works
**Inputs and Outputs:**
*   **Inputs:**
    *   `oTexCoords`: Smooth `vec3` texture coordinates representing the ray origin in texture space [S1, unofficial].
    *   `u_Volume3DSampler`: A `sampler3D` object accessing the volume data [S1, unofficial].
    *   `u_cameraPosition`: A `vec3` defining the camera position in texture space [S1, unofficial].
    *   `u_stepSize`: A `vec3` defining the uniform step size for ray marching [S1, unofficial].
*   **Outputs:**
    *   `FragColor`: A `vec4` storing the accumulated color (RGB) and opacity (A) [S1, unofficial].

**Data Structures:**
*   `FragColor`: Initialized to transparent black `(0.0, 0.0, 0.0, 0.0)` before the loop [G].
*   `dataPosition`: A `vec3` tracking the current position along the ray in texture coordinates [S1, unofficial].
*   `geometryDirection`: A `vec3` representing the normalized direction of the ray [S1, unofficial].
*   `directionStep`: A `vec3` representing the displacement vector for each step, calculated as `geometryDirection * u_stepSize` [S1, unofficial].

**Parameters:**
*   `MAX_SAMPLES`: Constant integer defining the maximum number of ray march steps per ray. Value: 300 [S1, unofficial].
*   Early ray termination threshold: 0.99 [S1, unofficial].

**Steps:**
1.  **Initialize Ray Origin:** Set `dataPosition` to `oTexCoords` [S1, unofficial].
2.  **Calculate Ray Direction:** Compute `geometryDirection` by subtracting 0.5 from `oTexCoords`, subtracting `u_cameraPosition`, and normalizing the result [S1, unofficial].
3.  **Calculate Step Vector:** Compute `directionStep` by multiplying `geometryDirection` with `u_stepSize` [S1, unofficial].
4.  **Loop:** Iterate `i` from 0 to `MAX_SAMPLES` (300) [S1, unofficial]:
    a.  **Advance Ray:** Advance `dataPosition` by `directionStep` [S1, unofficial].
    b.  **Check Boundary:** Check if the ray has exited the volume boundary. If `dot(sign(dataPosition - texMin), sign(texMax - dataPosition)) < 3.0`, break the loop [S1, unofficial].
    c.  **Sample Volume:** Sample the volume at `dataPosition` using `u_Volume3DSampler` to get `fSample` [S1, unofficial].
    d.  **Calculate Contribution:** Compute `prev_alpha` as `fSample - (fSample * FragColor.a)` [S1, unofficial].
    e.  **Update Color:** Update `FragColor.rgb` as `prev_alpha * vec3(fSample) + FragColor.rgb` [S1, unofficial].
    f.  **Update Alpha:** Update `FragColor.a` as `FragColor.a + prev_alpha` [S1, unofficial].
    g.  **Check Termination:** If `FragColor.a > 0.99`, break the loop [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def ray_casting_compositing_loop(oTexCoords, u_Volume3DSampler, u_cameraPosition, u_stepSize, texMin, texMax):
    # Minimal vector classes for valid Python execution
    class Vec3:
        def __init__(self, x, y, z): self.x, self.y, self.z = x, y, z
        def __sub__(self, other): return Vec3(self.x - other.x, self.y - other.y, self.z - other.z)
        def __add__(self, other): return Vec3(self.x + other.x, self.y + other.y, self.z + other.z)
        def __mul__(self, s): return Vec3(self.x * s, self.y * s, self.z * s)
    class Vec4:
        def __init__(self, r, g, b, a): self.r, self.g, self.b, self.a = r, g, b, a

    def sign(x): return 1.0 if x > 0.0 else (-1.0 if x < 0.0 else 0.0)
    def dot(v1, v2): return v1.x * v2.x + v1.y * v2.y + v1.z * v2.z
    def normalize(v):
        l = (v.x**2 + v.y**2 + v.z**2)**0.5
        return Vec3(v.x/l, v.y/l, v.z/l)
    def texture_lookup(sampler, pos):
        # Placeholder for actual 3D texture sampling; returns a scalar
        return 0.5 

    FragColor = Vec4(0.0, 0.0, 0.0, 0.0)
    dataPosition = oTexCoords
    raw_dir = oTexCoords - Vec3(0.5, 0.5, 0.5) - u_cameraPosition
    geometryDirection = normalize(raw_dir)
    directionStep = geometryDirection * u_stepSize

    for i in range(300):
        dataPosition = dataPosition + directionStep
        if dot(Vec3(sign(dataPosition.x - texMin.x), sign(dataPosition.y - texMin.y), sign(dataPosition.z - texMin.z)),
               Vec3(sign(texMax.x - dataPosition.x), sign(texMax.y - dataPosition.y), sign(texMax.z - dataPosition.z))) < 3.0:
            break
        fSample = texture_lookup(u_Volume3DSampler, dataPosition)
        prev_alpha = fSample - (fSample * FragColor.a)
        color_contrib = Vec3(fSample, fSample, fSample)
        FragColor.r = prev_alpha * color_contrib.x + FragColor.r
        FragColor.g = prev_alpha * color_contrib.y + FragColor.g
        FragColor.b = prev_alpha * color_contrib.z + FragColor.b
        FragColor.a += prev_alpha
        if FragColor.a > 0.99:
            break
    return FragColor
```

**Edge Cases and Pitfalls:**
*   **Boundary Intersection:** The boundary check `dot(sign(dataPosition - texMin), sign(texMax - dataPosition)) < 3.0` assumes the volume is axis-aligned in texture space [S1, unofficial]. If the volume is not axis-aligned or the texture coordinates are not normalized to `[0,1]`, this check may fail [S1, unofficial].
*   **Precision:** Floating-point precision errors in `dataPosition` accumulation may cause the ray to slightly overshoot the boundary, leading to sampling outside the volume. The boundary check helps mitigate this, but clamping the sample coordinates is a common safeguard [G].
*   **Transparent Voxels:** If voxels are fully transparent, the ray may traverse the entire volume without contributing to the color, leading to unnecessary computation [S3, unofficial].

## Subtopics

### [Wavelet-Assisted Volume Ray Casting](10893-wavelet-assisted-volume-ray-casting.md)
Wavelet-Assisted Volume Ray Casting is an acceleration algorithm for direct volume rendering that uses wavelet coefficients to detect local signal frequency and determine the appropriate adaptive sampling distance along each ray. It solves the computational bottleneck of standard ray casting by skipping samples in low-frequency regions while maintaining high sampling rates in high-frequency areas, avoiding the expensive on-the-fly reconstruction from the wavelet domain. Unlike [Accumulative Rendering](14504-accumulative-rendering.md) which focuses on compositing order, or [Template-based Ray-Casting](14511-template-based-ray-casting.md) which uses geometric templates, this method relies on a precomputed "index volume" to guide spatial traversal, offering a framework for controlled approximation of rendering quality.

### [Accumulative Rendering](14504-accumulative-rendering.md)
Accumulative rendering is a specialized rendering mode designed for scenarios where the input dataset is acquired incrementally, such as in live microscopy or intraoperative imaging. It solves the latency problem of conventional volume rendering by allowing immediate visualization of new data without reprocessing the entire volume. Unlike standard ray casting which requires the full dataset to be present before rendering can begin, accumulative rendering combines the previous rendering output directly with a newly acquired slice, limiting computational load to only two slices regardless of the total dataset size.

### [Template-based Ray-Casting](14511-template-based-ray-casting.md)
Template-based Ray-Casting is a hardware-accelerated direct volume rendering technique that optimizes memory access patterns by processing voxels in planar slices parallel to the image plane. It distinguishes itself from shear-warp methods by performing full tri-linear interpolation and supporting sub-pixel ray origins, thereby eliminating view-dependent artifacts during base-plane switches.

### [Resampling Point Shading](14923-resampling-point-shading.md)
Resampling Point Shading is a volume rendering technique that improves shading accuracy by applying the lighting model directly to resampling points using interpolated normal vectors and opacity-weighted color interpolation, thereby avoiding Mach band artifacts and interface color confusion. It sets itself apart from its sibling [Accumulative Rendering](14504-accumulative-rendering.md) and [Template-based Ray-Casting](14511-template-based-ray-casting.md) by calculating normals and lighting at the resampling point itself rather than interpolating pre-shaded vertex colors or lighting results, which preserves microstructure details and physical color consistency at media interfaces.

### Multi-Cast / Super-Sampling Ray Casting
Instead of sampling one ray per pixel, sample 4 rays per pixel to improve image quality [S3, unofficial]. Produces better image quality but takes four times longer to render [S3, unofficial].

### GPU-based VF-Ray
A hardware-based ray casting algorithm for tetrahedral meshes that implements VF-Ray on GPU using CUDA to achieve interactive performance while maintaining memory efficiency [S4]. It implements the VF-Ray algorithm using General Purpose computation on GPU (GPGPU) and the Compute Unified Device Architecture (CUDA) from nVidia [S4]. It restructures the original data structures of VF-Ray to improve performance and memory usage [S4]. The algorithm used only one-half of the memory usage of the original software VF-Ray version [S4]. Comparing the GPU implementation to other hardware-based ray casting algorithms, it used between three to ten times less memory [S4]. Compared to other recent ray casting hardware-based algorithms, this approach uses 77% to 90% less memory, allowing it to render datasets for which others failed due to lack of GPU memory [S4].

## Operating characteristics
*   **Time Complexity:** $O(N \cdot M)$, where $N$ is the number of pixels and $M$ is the number of samples per ray (MAX_SAMPLES) [S1, unofficial].
*   **Memory Complexity:** $O(V)$, where $V$ is the size of the volume data, stored in a 3D texture [S1, unofficial].
*   **Measured Performance:** Not given in the sources.
*   **Scaling:** Performance scales linearly with resolution and step count [S1, unofficial].
*   **Strengths:** Simple to implement, single-pass, suitable for real-time rendering on GPUs [S1, unofficial].
*   **Limitations:** Not given in the sources.
*   **When to use:** When simplicity and speed are prioritized over high-quality anti-aliasing or adaptive sampling [S1, unofficial].
*   **When not:** When high-quality anti-aliasing is required without significant performance cost, or when the volume has large transparent regions [S3, unofficial].

## Background
The method is described in the context of a volumetric render engine implementation [S1, unofficial].

## Sources
[S1] Volumetric Render Engine - GitHub, https://github.com/mikejernil/volumetric-render-engine, fetched 2026-10-27 (unofficial)
[S2] Volume MLS Ray Casting, https://miriah.github.io/publications/volume_mls.pdf, fetched 2026-10-29 (unofficial)
[S3] Introduction to volume rendering, https://slidetodoc.com/introduction-to-volume-rendering-presented-by-zvi-devir/, fetched 2026-10-29 (unofficial)
[S4] Memory Efficient GPU-Based Ray Casting for Unstructured Volume Rendering, https://doi.org/10.2312/vg.vg-pbg08.155-162, fetched 2026-11-05
