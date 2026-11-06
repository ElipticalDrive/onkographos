# OptiX Ray Generation & Traversal Setup

Also called: OptiX 7 Ray Generation, OptiX Ray Generation, Ray Generation Program

## Overview
This method implements the ray generation and initial traversal setup for volume rendering within the NVIDIA OptiX 7 API [S2, unofficial]. It computes pixel rays, initializes the random number generator (RNG) for anti-aliasing, and dispatches volume tracing via `optixTrace` [S2, unofficial]. It differs from generic ray generation methods by integrating specific OptiX 7 launch parameters, SBT indexing, and visibility masks to restrict traversal to volume geometry [S2, unofficial].

## How it works
**Inputs:**
- Frame buffer dimensions: width $w$, height $h$ [S1].
- Camera parameters: position $\mathbf{e}$, direction vector, horizontal basis $\mathbf{r}$, vertical basis $\mathbf{u}$, focal length $f$, rotation angles $\phi$ (phi) and $\theta$ (theta) [S1][S2, unofficial].
- Launch parameters: `frame_index`, `frame.size`, `camera` struct [S2, unofficial].
- OptiX constants: `VISIBILITY_MASK_VOLUME`, `MethodOptiX::RADIANCE_RAY_TYPE`, `MethodOptiX::RAY_TYPE_COUNT` [S2, unofficial].

**Outputs:**
- Ray origin $\mathbf{o}$ and direction $\mathbf{d}$ for each pixel [S1][S2, unofficial].
- Payload for `optixTrace` [S2, unofficial].

**Data Structures:**
- `RandomTEA`: RNG state initialized with `frame_index` and `fbIndex` [S2, unofficial].
- `vec2f`/`vec3f`: 2D and 3D floating-point vectors for screen coordinates and ray data [S2, unofficial].
- `uint32_t` payload: Packed pointer data for ray payload [S2, unofficial].

**Steps:**
1. **Thread Launch:** Launch a CUDA kernel with $w \times h$ threads [S1]. Threads are grouped into $8 \times 8$ tiled blocks, consisting of two 32-thread wraps computed as SIMD vectors [S1].
2. **Pixel Indexing:** Compute the framebuffer index: `fbIndex = ix + iy * frame.size.x` [S2, unofficial].
3. **RNG Initialization:** Initialize the RNG state: `rng_state = RandomTEA(frame_index, fbIndex)` [S2, unofficial].
4. **Screen Coordinates:** Determine 2D screen space coordinates [S1]. Compute normalized screen position: `screen = vec2f((float)ix + 0.5, (float)iy + 0.5) / vec2f(frame.size)` [S2, unofficial]. Convert to UV coordinates where top-left is $[-1, 1]$ and bottom-right is $[1, -1]$ [S1].
5. **Ray Direction Calculation:**
   - Compute initial ray direction $rd_0$: $rd_0 = [u, v, f]^T / \sqrt{u^2 + v^2 + f^2}$, where $u, v$ are the UV coordinates and $f$ is the focal length [S1].
   - Apply rotation matrices to orient the ray: $rd = R_y(\theta) R_z(\phi) rd_0$ [S1].
   - Rotation matrices are $R_y(\theta) = \begin{bmatrix} \cos\theta & 0 & \sin\theta \\ 0 & 1 & 0 \\ -\sin\theta & 0 & \cos\theta \end{bmatrix}$ and $R_z(\phi) = \begin{bmatrix} 1 & 0 & 0 \\ 0 & \cos\phi & -\sin\phi \\ 0 & \sin\phi & \cos\phi \end{bmatrix}$ [S1].
   - Alternatively, using camera basis vectors: $\mathbf{d} = \text{normalize}(\text{camera.direction} + (\text{screen}.x - 0.5) \text{camera.horizontal} + (\text{screen}.y - 0.5) \text{camera.vertical})$ [S2, unofficial].
6. **Ray Origin:** Set $\mathbf{o} = \text{camera.position}$ [S1][S2, unofficial]. Any position $p$ along the ray is computed as $p = \mathbf{o} + \mathbf{d} \cdot t$ [S1].
7. **Payload Packing:** Pack the payload for OptiX trace: `packPointer(&payload, u0, u1)` [S2, unofficial].
8. **Trace Dispatch:** Call `optixTrace` with visibility mask `OptixVisibilityMask(VISIBILITY_MASK_VOLUME)`, flags `OPTIX_RAY_FLAG_DISABLE_ANYHIT`, SBT offset `MethodOptiX::RADIANCE_RAY_TYPE`, stride `MethodOptiX::RAY_TYPE_COUNT`, and miss index `MethodOptiX::RADIANCE_RAY_TYPE` [S2, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def normalize(v):
    return (v[0], v[1], v[2]) / (v[0]**2 + v[1]**2 + v[2]**2)**0.5

class RandomTEA:
    def __init__(self, frame_index, fbIndex):
        pass

def packPointer(payload, u0, u1):
    pass

def ray_generation_kernel(ix, iy, frame_size, frame_index, camera, output):
    if ix >= frame_size[0] or iy >= frame_size[1]:
        return

    # Step 2: Pixel Indexing
    fbIndex = ix + iy * frame_size[0]

    # Step 3: RNG Initialization
    rng_state = RandomTEA(frame_index, fbIndex)

    # Step 4: Screen Coordinates
    screen_x = (ix + 0.5) / frame_size[0]
    screen_y = (iy + 0.5) / frame_size[1]

    # Step 5: Ray Direction (Camera basis method)
    ray_dir = camera.direction + (screen_x - 0.5) * camera.horizontal + (screen_y - 0.5) * camera.vertical
    ray_dir = normalize(ray_dir)

    # Step 6: Ray Origin
    ray_org = camera.position

    # Step 7: Payload Packing
    payload = [0, 0]
    packPointer(payload, payload[0], payload[1])

    # Step 8: Trace Dispatch
    # optixTrace(ray_org, ray_dir, payload, VISIBILITY_MASK_VOLUME,
    #            OPTIX_RAY_FLAG_DISABLE_ANYHIT, MethodOptiX::RADIANCE_RAY_TYPE,
    #            MethodOptiX::RAY_TYPE_COUNT, MethodOptiX::RADIANCE_RAY_TYPE) [S2, unofficial]
    output[fbIndex] = (ray_org, ray_dir, payload)
```

**Edge Cases and Pitfalls:**
- Boundaries: Threads outside the frame buffer dimensions must be guarded [S1].
- Degenerate Input: Not given in the sources.

## Subtopics

### [Sky Gradient](1262-sky-gradient.md)
Generates rays for sky background. Choose when rendering scenes with skyboxes.

### [Spiral Camera Path Generation](1635-spiral-camera-path-generation.md)
Generates camera poses along a spiral. Choose for 360-degree volume visualization.

### [Arcball Camera](1639-arcball-camera.md)
Interactive camera control. Choose for user-driven exploration.

### [Uniform Sphere Sampling](1669-uniform-sphere-sampling.md)
Samples directions uniformly on a sphere. Choose for environment lighting.

### [Ray Origin and Direction Setup](1793-ray-origin-and-direction-setup.md)
Generic ray setup. Choose when not using OptiX.

### [First-Person Camera Control](2721-first-person-camera-control.md)
FPS-style camera. Choose for interactive volume navigation.

### [Arbitrary Source-Detector Geometry Projector](2975-arbitrary-source-detector-geometry-projector.md)
CT-style projection. Choose for medical imaging simulations.

### [Camera Class Structure](3022-camera-class-structure.md)
Data structure for camera parameters. Choose for organizing camera state.

### [Sky Radiance Model](3256-sky-radiance-model.md)
Models sky brightness. Choose for realistic sky illumination.

### [Compute Shader Ray Marching Setup](3487-compute-shader-ray-marching-setup.md)
GPU-based ray marching without OptiX. Choose for portable CUDA implementations.

### [Grid Ray Sampler](3930-grid-ray-sampler.md)
Samples rays on a grid. Choose for structured sampling patterns.

### [Elongated Structure Camera Trajectory Generation](4666-elongated-structure-camera-trajectory-generation.md)
Camera paths for long objects. Choose for scanning elongated volumes.

### [Fisheye Distortion Correction](4724-fisheye-distortion-correction.md)
Corrects lens distortion. Choose for wide-angle camera models.

### [Camera Matrix and Coordinate Transformation](6231-camera-matrix-and-coordinate-transformation.md)
Matrix-based camera transforms. Choose for complex coordinate systems.

### [Depth of Field and Lens Effects](6232-depth-of-field-and-lens-effects.md)
Simulates lens blur. Choose for realistic depth effects.

## Operating characteristics
- **Time Complexity:** $O(w \times h)$ for ray generation, where $w, h$ are image dimensions [S1].
- **Memory Complexity:** $O(1)$ per thread for local state [S1].
- **Performance:** Depends on GPU architecture and OptiX acceleration structure traversal efficiency [S2, unofficial].
- **Strengths:** Efficient GPU parallelism, hardware-accelerated ray traversal via OptiX [S2, unofficial].
- **Limitations:** Not given in the sources.
- **When to Use:** Not given in the sources.
- **When Not to Use:** Not given in the sources.

## Background
Originated in the open-volume-renderer project by VIDILabs, which implements an OptiX-7 renderer [S2, unofficial]. Used for volume rendering tasks within the open-volume-renderer framework [S2, unofficial].

## Sources
[S1] Raymarching Distance Fields with CUDA, https://www.mdpi.com/2079-9292/10/22/2730, fetched 2026-10-27
[S2] GitHub - VIDILabs/open-volume-renderer: Open source modular multi ..., https://github.com/VIDILabs/open-volume-renderer, fetched 2026-10-29 (unofficial)
