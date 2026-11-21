# Camera Ray Generation

Also called: unproject, Ray Unprojection

## Overview
Camera Ray Generation computes the 3D world-space origin and direction vectors for viewing rays corresponding to each pixel in a raster image, given explicit camera intrinsic and extrinsic parameters [S5, S9, S12, unofficial]. It solves the problem of mapping 2D screen coordinates to 3D world-space rays for volume rendering or ray tracing [S5, S9, unofficial]. Unlike [Pinhole Camera Ray Generation](5058-pinhole-camera-ray-generation.md), which often derives rays directly from geometric primitives such as field of view (FOV) and camera basis vectors, this method relies on explicit intrinsic ($K$) and extrinsic ($c2w$) matrices to define the projection geometry [S1, S2, S9, unofficial]. It builds on [Ray Sampling and Generation](15460-ray-sampling-and-generation.md) by providing the specific geometric unprojection logic required to initialize the parallel execution model [G].

## How it works
**Inputs:**
*   Image dimensions: width $W$, height $H$ [S4, S5, unofficial].
*   Pixel coordinates: integer grid $(i, j)$ where $0 \le i < W$ and $0 \le j < H$ [S4, S5, unofficial].
*   Camera Intrinsics: $3 \times 3$ matrix $K$ containing focal lengths ($f_x, f_y$) and principal point ($c_x, c_y$) [S5, S9, S12, unofficial].
*   Camera Extrinsic: $4 \times 4$ matrix $c2w$ (camera-to-world) containing rotation $R$ and translation $T$ [S5, S9, S12, unofficial].
*   Optional: Sub-pixel offsets $(s_x, s_y)$ for anti-aliasing [S12, unofficial].
*   Optional: Depth values $z$ if unprojecting specific 3D points rather than directions [S9, unofficial].

**Outputs:**
*   Ray Origins: Tensor of shape $(H, W, 3)$ or $(N, 3)$ representing world-space positions [S5, S9, S17, unofficial].
*   Ray Directions: Tensor of shape $(H, W, 3)$ or $(N, 3)$ representing normalized world-space vectors [S5, S9, S17, unofficial].

**Data Structures:**
*   `RayBundle` or `Rays` namedtuple/struct containing fields: `origins` (float32), `directions` (float32), `viewdirs` (float32) [S17, S19, unofficial].
*   In some implementations, a `Ray` struct contains `origin` and `direction` (normalized) [S15, unofficial].

**Steps:**

1.  **Generate Pixel Coordinates:** Create a meshgrid of pixel indices $i$ (width) and $j$ (height) [S4, S5, unofficial].
    *   $i \in [0, W-1]$, $j \in [0, H-1]$ [S4, unofficial].
    *   Apply flips if specified (e.g., `flip_x`, `flip_y`) to match coordinate system conventions [S5, unofficial].
    *   Adjust coordinates based on mode: 'center' adds 0.5 to pixel centers, 'random' adds uniform noise [S4, unofficial].

2.  **Compute Normalized Device Coordinates (NDC) or Camera Space Coordinates:**
    *   Method A (Matrix-based): Homogenize 2D coordinates by appending a 1.0 component to form vectors $\mathbf{p}_{2d} = [i, j, 1]^T$ [S9, unofficial].
    *   Method B (Geometric): Compute aspect-corrected UV coordinates in $[-1, 1]$ with y-up:
        $$ \text{uv}_x = \frac{i + s_x + 0.5}{W} \cdot 2 - 1 $$
        $$ \text{uv}_y = \frac{j + s_y + 0.5}{H} \cdot 2 - 1 $$
        $$ \text{uv}_x \leftarrow \text{uv}_x \cdot \frac{W}{H} $$
        [S8, S12, unofficial]. Note: Vulkan's flipped Y-axis in clip space may require negating Y during reconstruction [S14, unofficial].

3.  **Unproject to Camera Space:**
    *   Using Intrinsics $K$: Multiply by the inverse intrinsic matrix to get ray directions in camera space $\mathbf{d}_{cam}$ [S9, unofficial].
        $$ \mathbf{d}_{cam} = K^{-1} \begin{bmatrix} i - c_x \\ j - c_y \\ f \end{bmatrix} $$
        Where $f$ is typically 1.0 for direction normalization, or derived from focal length [S9, S13, unofficial].
    *   Alternatively, using FOV and Basis Vectors:
        $$ \mathbf{d}_{cam} = \text{normalize}(\text{uv}_x \cdot \mathbf{u}_{right} + \text{uv}_y \cdot \mathbf{u}_{up} - \mathbf{u}_{forward}) $$
        [S8, S12, unofficial]. Here $\mathbf{u}_{right}, \mathbf{u}_{up}, \mathbf{u}_{forward}$ are the camera basis vectors.

4.  **Normalize Directions:**
    *   Normalize the camera-space direction vectors to unit length:
        $$ \mathbf{d}_{cam} \leftarrow \frac{\mathbf{d}_{cam}}{\|\mathbf{d}_{cam}\|} $$
        [S9, S17, unofficial].

5.  **Transform to World Space:**
    *   Rotate the camera-space directions using the rotation part of the extrinsic matrix $R$ (top-left $3 \times 3$ of $c2w$) [S5, S17, unofficial].
        $$ \mathbf{d}_{world} = R \cdot \mathbf{d}_{cam} $$
        [S5, S17, unofficial].
    *   Extract the ray origin from the translation part of the extrinsic matrix $T$ (last column of $c2w$) [S5, S9, unofficial].
        $$ \mathbf{o}_{world} = T $$
        [S5, S9, unofficial]. Broadcast $\mathbf{o}_{world}$ to match the shape of $\mathbf{d}_{world}$ [S17, unofficial].

6.  **Optional: Depth Unprojection:**
    *   If 3D points are required instead of directions, scale the unprojected camera-space directions by the depth $z$:
        $$ \mathbf{p}_{cam} = \mathbf{d}_{cam} \cdot z $$
        [S9, unofficial].
    *   Transform $\mathbf{p}_{cam}$ to world space using $c2w$ [S9, unofficial].

7.  **Front-of-Camera Check:**
    *   Verify that points in camera space have $z > -\epsilon$ (where $\epsilon = 10^{-6}$) to ensure they are in front of the camera [S9, unofficial].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import numpy as np

def generate_camera_rays(W, H, K, c2w, subpixel_offsets=None, mode='center'):
    """
    Generate camera rays for volume rendering.
    
    Args:
        W: Image width.
        H: Image height.
        K: 3x3 intrinsic matrix [[fx, 0, cx], [0, fy, cy], [0, 0, 1]].
        c2w: 4x4 camera-to-world transformation matrix.
        subpixel_offsets: Optional tuple (sx, sy) for anti-aliasing.
        mode: 'center' (pixel centers), 'lefttop' (corners), 'random' (jitter).
        
    Returns:
        origins: (H, W, 3) array of world-space ray origins.
        directions: (H, W, 3) array of normalized world-space ray directions.
    """
    # 1. Generate Pixel Coordinates
    i, j = np.meshgrid(np.arange(W), np.arange(H), indexing='ij')
    
    # 2. Adjust for mode and subpixel offsets
    if mode == 'center':
        i = i.astype(np.float32) + 0.5
        j = j.astype(np.float32) + 0.5
    elif mode == 'random':
        i = i.astype(np.float32) + np.random.uniform(0, 1, size=(H, W))
        j = j.astype(np.float32) + np.random.uniform(0, 1, size=(H, W))
    
    if subpixel_offsets is not None:
        sx, sy = subpixel_offsets
        i += sx
        j += sy
        
    # 3. Unproject to Camera Space using Intrinsics
    # Extract intrinsic parameters
    fx = K[0, 0]
    fy = K[1, 1]
    cx = K[0, 2]
    cy = K[1, 2]
    
    # Compute normalized coordinates relative to principal point
    # Note: Coordinate system conventions vary. This assumes standard pinhole model.
    # If inverse_y is true, j is flipped.
    u = (i - cx) / fx
    v = (j - cy) / fy
    
    # Construct direction vectors in camera space (z=1 plane)
    # Shape: (H, W, 3)
    directions_cam = np.stack([u, v, np.ones_like(u)], axis=-1)
    
    # 4. Normalize Directions
    norms = np.linalg.norm(directions_cam, axis=-1, keepdims=True)
    # Avoid division by zero
    norms = np.maximum(norms, 1e-6)
    directions_cam = directions_cam / norms
    
    # 5. Transform to World Space
    # Extract rotation and translation from c2w
    R = c2w[:3, :3]
    T = c2w[:3, 3]
    
    # Rotate directions: (H, W, 3) @ (3, 3) -> (H, W, 3)
    # Using einsum or matmul
    directions_world = np.einsum('ijk,kl->ijl', directions_cam, R)
    
    # 6. Set Origins
    # Broadcast translation T to (H, W, 3)
    origins_world = np.broadcast_to(T, (H, W, 3))
    
    return origins_world, directions_world
```

**Parameters with Typical Values:**
*   $K$: Depends on camera calibration. Typical $f_x, f_y \approx 500-1000$ pixels for standard lenses [S16, unofficial].
*   $c2w$: Identity for world-aligned cameras; varies for tracked cameras [S5, unofficial].
*   $\epsilon$: $10^{-6}$ for front-of-camera check [S9, unofficial].

**Edge Cases and Pitfalls:**
*   **Coordinate System Mismatch:** The 3D texture/model may be centered at different coordinates than the camera's reference frame, requiring remapping of camera movements into voxel coordinates [S7, unofficial].
*   **Vulkan Y-Axis:** Vulkan's flipped Y-axis in clip space requires negating Y during view-space reconstruction [S14, unofficial].
*   **Degenerate Matrices:** Check for zero viewport height or singular intrinsic matrices to avoid NaNs [S15, unofficial].
*   **Aspect Ratio:** Incorrect aspect ratio correction leads to distorted FOV [S8, S15, unofficial].

## Subtopics

### [Orthographic (Parallel) Camera Ray Generation](1319-orthographic-parallel-camera-ray-generation.md)
Orthographic (Parallel) Camera Ray Generation constructs initial rays for volume rendering by defining a projection plane parallel to the scanline and casting rays in the viewing direction within that plane. It serves as the geometric foundation for orthographic ray casting, where ray origins are distributed across a 2D grid and all rays share a single normalized direction. This method differs from perspective ray generation by maintaining parallel rays, eliminating vanishing points and depth-dependent field-of-view scaling.

### [Sample Streaming Payload Ray Computation](1509-sample-streaming-payload-ray-computation.md)
Sample Streaming Payload Ray Computation regenerates ray geometry (origin and direction) from stored pixel indices and camera parameters, enabling sample streaming and resumption in volume rendering pipelines. It maps pixel coordinates to Normalized Device Coordinates (NDC) and unprojects them to world space to compute unit-length direction vectors. Unlike static ray generation, this method is designed for payloads that persist across frames or batches, allowing rays to be reconstructed on-demand from minimal state.

### [OptiX Ray Generation & Traversal Setup](1552-optix-ray-generation-traversal-setup.md)
This method implements the ray generation and initial traversal setup for volume rendering within the NVIDIA OptiX 7 API. It computes pixel rays, initializes the random number generator (RNG) for anti-aliasing, and dispatches volume tracing via `optixTrace`. It differs from generic ray generation methods by integrating specific OptiX 7 launch parameters, SBT indexing, and visibility masks to restrict traversal to volume geometry.

### [Object Space Ray Setup](1642-object-space-ray-setup.md)
Object Space Ray Setup constructs ray origins and directions within the local coordinate system of a mesh, ensuring that the raytraced volume moves, scales, and rotates rigidly with the object's transform. Unlike world-space ray generation, which treats the volume as a static window into the world, this method embeds the object's transformation directly into the ray parameters, allowing the raytraced geometry to behave like a physical part of the mesh. It serves as the foundational step for [Camera Ray Generation](7456-camera-ray-generation.md) when the rendering target is a specific object rather than the entire scene.

### [Custom Frustum Ray Generation via Vertex Z-Index](1753-custom-frustum-ray-generation-via-vertex-z-index.md)
This method generates per-pixel ray directions for volume rendering by encoding the index of pre-computed frustum corner vectors into the Z-coordinate of a full-screen quad's vertices, allowing the vertex shader to look up the correct ray direction from a uniform array. It differs from [Camera Matrix Ray Generation](2172-camera-matrix-ray-generation.md) by avoiding per-fragment matrix multiplications for direction calculation, instead using a discrete lookup based on vertex attributes. It differs from [Basic GPU Ray Generation via Fragment Shader](2208-basic-gpu-ray-generation-via-fragment-shader.md) by performing the direction selection in the vertex shader rather than the fragment shader, reducing redundant calculations for adjacent pixels sharing the same vertex.

### [Ray Origin Offset Noise](2118-ray-origin-offset-noise.md)
Ray Origin Offset Noise is a technique that perturbs the starting position of a ray march to avoid banding artifacts. It differs from its sibling [Orthographic (Parallel) Camera Ray Generation](1319-orthographic-parallel-camera-ray-generation.md) by adding stochastic variation to the ray origin rather than strictly adhering to pixel centers, and from [Sample Streaming Payload Ray Computation](1509-sample-streaming-payload-ray-computation.md) by focusing on initialization jitter rather than payload management during traversal [G].

### [Camera Matrix Ray Generation](2172-camera-matrix-ray-generation.md)
Camera Matrix Ray Generation computes the 3D world-space origin and direction vectors for viewing rays corresponding to each pixel in a raster image, given explicit camera intrinsic and extrinsic parameters. It solves the problem of mapping 2D screen coordinates to 3D world-space rays for volume rendering or ray tracing. Unlike [Pinhole Camera Ray Generation](5058-pinhole-camera-ray-generation.md), which often derives rays directly from geometric primitives such as field of view (FOV) and camera basis vectors, this method relies on explicit intrinsic ($K$) and extrinsic ($c2w$) matrices to define the projection geometry.

### [Basic GPU Ray Generation via Fragment Shader](2208-basic-gpu-ray-generation-via-fragment-shader.md)
This method generates camera rays for ray tracing by rendering a single quad to the screen, allowing the fragment shader to compute the ray origin and direction for each pixel using interpolated vertex coordinates. It differs from [Camera Matrix Ray Generation](2172-camera-matrix-ray-generation.md) by avoiding explicit matrix multiplications in favor of geometric basis vectors derived from camera parameters, and from [Orthographic (Parallel) Camera Ray Generation](1319-orthographic-parallel-camera-ray-generation.md) by supporting perspective projection through a viewing plane distance.

### [Ray Position Calculation](2895-ray-position-calculation.md)
Ray Position Calculation computes the 3D spatial coordinates of sample points (query points) along a bundle of rays defined by their origins and directions, constrained within a specific depth range. It solves the problem of mapping 1D depth samples to 3D Euclidean space for volume rendering or neural rendering pipelines. It differs from [Camera Ray Generation](7456-camera-ray-generation.md) by assuming the ray origins and directions are already computed; it focuses solely on the linear interpolation along the ray vector to determine sample locations.

### [Epipolar Ray Intersection](7644-epipolar-ray-intersection.md)
Epipolar Ray Intersection lifts 2D image points to 3D space by intersecting rays constructed from those points with a set of input world-space rays. It solves the problem of finding the precise 3D positions and depths corresponding to specified 2D coordinates on epipolar lines defined by camera origins and directions. It differs from sibling methods like [Orthographic (Parallel) Camera Ray Generation](1319-orthographic-parallel-camera-ray-generation.md) by explicitly handling the intersection of two distinct ray sets (image-space and world-space) rather than generating rays from a single camera model, and from [Sample Streaming Payload Ray Computation](1509-sample-streaming-payload-ray-computation.md) by focusing on geometric reprojection via epipolar constraints rather than payload streaming.

### [Flattened Ray Generation for Training](8180-flattened-ray-generation-for-training.md)
Flattened Ray Generation for Training is a data preparation method that converts a batch of training images into a single, flat tensor of rays and corresponding pixel colors, optimized for stochastic gradient descent in neural volume rendering. Unlike [Camera Ray Generation](7456-camera-ray-generation.md), which generates rays on-the-fly per batch or view, this method pre-aggregates rays from all training poses into a unified structure, enabling efficient random sampling of individual rays across the entire dataset during training. It differs from [Precomputed Ray Buffering](#precomputed-ray-buffering) by typically handling the flattening and indexing logic within the training loop or a dedicated sampler rather than storing the entire dataset in CPU RAM, though both aim to decouple ray generation from the forward pass.

### PathCamera
A camera that glides forward along a procedural road centerline, yawing to follow bends and banking gently, producing an orthonormal ray basis for the shader [S8, unofficial].
*   **Steps:** Update position $z += \text{speed} \cdot dt$; Compute eye position $[roadX(z), \text{height}, z]$; Compute target $[roadX(z + \text{lookAhead}), \text{lookHeight}, z + \text{lookAhead}]$; Compute forward $\text{normalize}(\text{target} - \text{position})$; Compute right $\text{normalize}(\text{cross}(\text{forward}, \text{WORLD\_UP}))$; Compute up $\text{normalize}(\text{cross}(\text{right}, \text{forward}))$ [S8, unofficial].
*   **Parameters:** Eye height: 3.5 world units; Look-ahead distance: 15 world units; Look height: 1.5 world units; World up vector: $[0, 1, 0]$ [S8, unofficial].

### PinholeRayTracer Configuration
Camera orientation convention: Z-axis along optical axis (orthogonal to image plane), X-axis pointing towards bottom/ground, Y-axis according to right-hand rule [S18, unofficial].
*   **Parameters:** RES: 0.02f; RANGE_MAX: 100000.0f; HFOV: $M\_PI \cdot 160.0f / 180.0f$; VFOV: $M\_PI \cdot 160.0f / 180.0f$; HSIZE: 6; VSIZE: 4 [S18, unofficial].

### Spherical Ray Casting
Generates rays by creating a meshgrid of spherical coordinates $\theta$ and $\phi$ [S19, unofficial].
*   **Steps:** Create linearly spaced $\theta \in [0, 2\pi]$ and $\phi \in [0, \pi]$; Compute directions in camera frame (y is up): $[-\sin\phi \sin\theta, \cos\phi, \sin\phi \cos\theta]$; Transform to world space using camera rotation matrix; Compute offset directions dx and dy; Crop directions to remove extra row/column; Compute origins by broadcasting camera translation; Compute radii using norms of dx and dy; Set imageplane to zeros [S19, unofficial].
*   **Layout:** Returns `utils.Rays` with lossmult=1, near/far values, and cam_idx=0 [S19, unofficial].

### 2D Eraser
A 2D sculpting tool that removes voxels inside a selection volume defined by perspective projection from the user's point of view through a circular cursor on the view plane [S20].
*   **Steps:** Set cursor as a circle aligned with the view plane; Set virtual pointer origin to user's point of view; Set direction vector from point of view to cursor center; For each voxel, project its center onto the view plane; Calculate 2D Euclidean distance $d$ from projected voxel center $V$ to cursor center $P$; If $d < r$, remove the voxel [S20].
*   **Formula:** Selection condition: $d < r$, where $d$ is the 2D Euclidean distance on the view plane between the projected voxel center and cursor center $P$, and $r$ is the cursor radius [S20].
*   **Pitfall:** Perspective projection causes the selection volume to be a cone with apex at the point of view, making the tool dependent on viewing parameters. Orthographic projection yields a cylinder, which is preferred for 3D tools to detach from the point of view [S20].

### Ray Extraction
Selects a point on a plane surface of the volume cube aligned with two of the three primary axes (e.g., Y and Z) and casts a ray down to the third axis (e.g., X-axis) to determine the ray path [S21].
*   **Steps:** Derive four ray representations: 1) An image pair co-planar to the ray in the other two axes (e.g., X-Z and X-Y planes). 2) An intensity profile along the ray. 3) The corresponding SOI label profile. 4) Location coordinates of entry and exit points [S21].
*   **Parameters:** Number of rays: 64 rays per axis, 192 total per volume [S21].

### Front of Camera Check
Checks if 3D points in camera space are in front of the camera ($Z > -\epsilon$) [S9, unofficial].
*   **Steps:** Check if the Z coordinate (last element) is $> -\epsilon$: return $\text{xyz}[..., -1] > -\epsilon$ [S9, unofficial].
*   **Parameters:** $\epsilon$: float = $10^{-6}$ (tolerance for front-of-camera check) [S9, unofficial].

### Voxel Picking
Press the right mouse button to select the line of sight through the cursor [S22]. Draw a signal corresponding to the values of voxels along this line at the bottom of the screen [S22].

## Operating characteristics
*   **Time Complexity:** $O(W \cdot H)$ for generating rays for all pixels, where $W$ and $H$ are image dimensions [S4, S5, unofficial]. Each ray computation is $O(1)$ involving matrix-vector multiplications [S9, unofficial].
*   **Memory Complexity:** $O(W \cdot H \cdot 3)$ to store origins and directions [S5, S9, unofficial].
*   **Performance:** Highly parallelizable on GPUs due to the independent nature of each pixel's ray generation [G]. Performance is limited by memory bandwidth when storing large ray buffers [S17, unofficial].
*   **Strengths:** Flexible camera models via explicit matrices; supports arbitrary camera movements and poses [S7, S9, unofficial].
*   **Limitations:** Requires accurate camera calibration (intrinsics/extrinsics); sensitive to coordinate system mismatches [S7, S14, unofficial].
*   **When to Use:** When camera parameters are known from calibration or tracking (e.g., NeRF, SLAM outputs) [S5, S9, unofficial].
*   **When Not to Use:** When simple geometric primitives (e.g., fixed FOV) are sufficient and matrix overhead is undesirable [S12, unofficial].

## Background
Camera Ray Generation is a fundamental technique in computer graphics, originating from the need to map rasterized screen coordinates to 3D space for ray tracing and volume rendering [S5, S9, unofficial]. It is widely used in neural radiance fields (NeRF) and other view synthesis methods where camera poses are estimated from images [S3, S9, S16, unofficial]. The method relies on the pinhole camera model and its matrix representation, which has been standard since the early days of computer vision and graphics [S12, S14, unofficial].

## Sources
[S1] GitHub - razzle-reinizu/Voxel-World-Shader: A GLSL voxel-world shader ..., https://github.com/razzle-reinizu/Voxel-World-Shader, fetched 2026-10-25 (unofficial)
[S2] xwghua/clearvolume-plus, https://github.com/xwghua/clearvolume-plus, fetched 2026-10-26 (unofficial)
[S3] autonomousvision/sdfstudio, https://github.com/autonomousvision/sdfstudio, fetched 2026-10-26 (unofficial)
[S4] sjtuytc/UnboundedNeRFPytorch, https://github.com/sjtuytc/UnboundedNeRFPytorch, fetched 2026-10-26 (unofficial)
[S5] Jumpat/SegmentAnythingin3D, https://github.com/Jumpat/SegmentAnythingin3D, fetched 2026-10-26 (unofficial)
[S6] KaanAres10/Volume-Rendering, https://github.com/KaanAres10/Volume-Rendering, fetched 2026-10-26 (unofficial)
[S7] Display voxels using shaders in openGL, https://stackoverflow.com/questions/24394580/display-voxels-using-shaders-in-opengl, fetched 2026-10-27 (unofficial)
[S8] PIGJET/neon-raymarcher, https://github.com/PIGJET/neon-raymarcher, fetched 2026-10-27 (unofficial)
[S9] cvg/NoPoSplat, https://github.com/cvg/NoPoSplat, fetched 2026-10-27 (unofficial)
[S10] Ne0nWinds/WebGPU-Volumetrics-Demo, https://github.com/Ne0nWinds/WebGPU-Volumetrics-Demo, fetched 2026-10-27 (unofficial)
[S11] ziplab/VolSplat, https://github.com/ziplab/VolSplat, fetched 2026-10-27 (unofficial)
[S12] SRaimondi/VolumeRender, https://github.com/SRaimondi/VolumeRender, fetched 2026-10-27 (unofficial)
[S13] aivv73/bend-voxel, https://github.com/aivv73/bend-voxel, fetched 2026-10-28 (unofficial)
[S14] migaelstrydom/VoxelPhase, https://github.com/migaelstrydom/VoxelPhase, fetched 2026-10-28 (unofficial)
[S15] Implement Visualization/RayCaster, https://github.com/rp-bot/ASE-final-project/issues/54, fetched 2026-10-29 (unofficial)
[S16] GitHub - apchenstu/TensoRF: [ECCV 2022] Tensorial Radiance Fields, a ..., https://github.com/apchenstu/TensoRF, fetched 2026-10-29 (unofficial)
[S17] https://github.com/CPREgroup/CaSE-NeRF, https://github.com/CPREgroup/CaSE-NeRF, fetched 2026-10-30 (unofficial)
[S18] RMonica/basic_next_best_view, https://github.com/RMonica/basic_next_best_view, fetched 2026-10-24 (unofficial)
[S19] google-research/multinerf, https://github.com/google-research/multinerf, fetched 2026-10-26 (unofficial)
[S20] Volume sculpting based on geometric tools, https://doi.org/10.1007/bf03194498, fetched 2026-10-26
[S21] A Transfer Function Design for Medical Volume Data Using a Knowledge Database Based on Deep Image and Primitive Intensity Profile Features Retrieval, https://doi.org/10.1007/s11390-024-3419-7, fetched 2026-10-27
[S22] Appendix A. Volume Rendering Examples, https://techpubs.jurassic.nl/library/manuals/3000/007-3720-002/sgi_html/apa.html, fetched 2026-10-30
