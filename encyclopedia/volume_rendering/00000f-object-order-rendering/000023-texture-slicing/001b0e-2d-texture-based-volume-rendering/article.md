# 2D Texture-Based Volume Rendering

Also called: Object-aligned slice projection, Proxy geometry with 2D textures, Texture-based VR with 2D textures, VR Basado en Texturas 2D, View-Aligned Slice Rendering, VolumeRover2, cvc::volslice

## Overview
2D Texture-Based Volume Rendering is a GPU-accelerated object-order technique that represents volumetric data as stacks of 2D textures, projecting them onto axis-aligned proxy geometry and compositing them back-to-front using hardware blending [S1, S3]. Unlike 3D texture methods that sample a single trilinearly interpolated volume, this approach relies on bilinear interpolation of 2D slices, making it historically significant for early real-time rendering but prone to specific interpolation artifacts [S1, S3].

## How it works
**Inputs and Outputs:**
*   **Input:** A 3D scalar volume dataset (dimensions $D_x \times D_y \times D_z$), a camera view matrix (local-to-clip), and a transfer function (color/opacity mapping) [S1, S2, S3].
*   **Output:** A 2D image with accumulated color and opacity [S1, S3].

**Data Structures:**
1.  **2D Texture Stacks:** The volume is decomposed into three sets of 2D textures, one for each axis (X, Y, Z) [S3].
    *   **Format:** `GL_LUMINANCE` (stores single-component opacity) [S3]. Note: `GL_INTENSITY` is unsupported on some legacy hardware [S3].
    *   **Dimensions:** Power-of-two padding. If the original volume dimension is $D$, the texture width/height is the smallest power of two $P \ge D$ [S3].
    *   **Content:** Pre-classified opacity values. Color is applied during rendering via multiplication with a user-defined tissue color [S3].
    *   **Total Textures:** $N_{total} = D_x + D_y + D_z$ [S3].
    *   **Memory:** $P^2 \times N_{total}$ bytes (assuming 1 byte per texel) [S3].
2.  **Proxy Geometry:** Axis-aligned quadrilaterals (or triangle fans) representing the slices [S3].
    *   **Texture Coordinates:** Computed in the vertex shader as an affine map of model coordinates (`vertexMC`), eliminating the need for a separate texture coordinate array [S2].
    *   **Clipping:** Generated using a 256-case plane/cube clipping table to handle intersections with the volume bounding box [S2].
3.  **Transfer Function:** A 256x1 RGBA Look-Up Table (LUT) [S2].

**Numbered Steps:**
1.  **Preparation (Offline/Initialization):**
    *   For each axis $A \in \{X, Y, Z\}$:
        *   Extract 2D slices perpendicular to $A$ [S3].
        *   Pad each slice to power-of-two dimensions ($P \times P$) [S3].
        *   Upload to GPU as `GL_LUMINANCE` textures [S3].
2.  **Render Loop (Per Frame):**
    *   **Axis Selection:** Calculate the angle between the volume's principal axes and the projection plane (or camera view direction). Select the axis $A_{best}$ with the smallest angle [S3].
    *   **Slice Count Calculation:** Compute the number of slices $N$ to render:
        $$ N = 2 \cdot (10 + \text{max\_planes} \cdot q^3) $$
        where $\text{max\_planes}$ is the maximum allowed planes and $q$ is a quality factor [S2].
    *   **Cap:** Limit $N$ to $10 \cdot \text{max\_planes}$ per frame [S2].
    *   **Sweep Generation:**
        *   Compute slice positions based on the camera's local-to-clip matrix and the volume's local bounding box [S2].
        *   Perform the sweep in "legacy ratio space" (normalized coordinates) to preserve plane parallelism and spacing, then map to the caller's box via uniform scale and translation [S2].
        *   Generate triangle-fan polygons for each slice using the 256-case plane/cube clipping table [S2].
        *   Order polygons back-to-front relative to the camera [S2].
    *   **Culling/Clipping:**
        *   **Near-plane peel:** Dim or cull slices closer than a specific fraction of the diagonal to the near plane [S2].
        *   **Sub-cube crop:** Clip texture coordinates to valid volume bounds to prevent sampling outside the volume [S2].
    *   **Rendering:**
        *   Bind the selected axis's texture stack [S3].
        *   Set blend state: `SRC_ALPHA / ONE_MINUS_SRC_ALPHA`, depth writing disabled [S2].
        *   For each slice polygon (back-to-front):
            *   Vertex Shader: Compute texture coordinates from vertex model coordinates [S2].
            *   Fragment Shader: Sample the 2D texture [S3]. Multiply the sampled luminance (opacity) by the user-defined tissue color [S3].
            *   Hardware Blending: Accumulate color and opacity into the framebuffer [S3].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def render_2d_texture_volume(volume_data, camera_matrix, transfer_function, max_planes, quality):
    # Inputs: volume_data (Dx, Dy, Dz), camera_matrix (4x4 local-to-clip), 
    # transfer_function (256x4 RGBA LUT), max_planes (int), quality (float)
    # Assumes textures_X, textures_Y, textures_Z are pre-uploaded as P x P GL_LUMINANCE textures
    
    # 1. Axis Selection
    view_vector = normalize([-camera_matrix[0][2], -camera_matrix[1][2], -camera_matrix[2][2]])
    axes = [[1, 0, 0], [0, 1, 0], [0, 0, 1]]
    angles = [1.0 - dot(axis, view_vector) for axis in axes]
    best_axis = ['X', 'Y', 'Z'][argmin(angles)]
    
    # 2. Determine Slice Count
    N = 2 * (10 + max_planes * (quality ** 3))
    N = min(N, 10 * max_planes)
    
    # 3. Generate Slices
    # Compute slice positions in legacy ratio space, then map to caller's box
    slice_positions = [i / N for i in range(N)] # Legacy ratio space positions
    polygons = []
    for pos in slice_positions:
        # Generate triangle-fan polygon using 256-case plane/cube clipping table [S2]
        # Order polygons back-to-front relative to camera [S2]
        polygons.append(generate_fan_polygon(pos, volume_box))
    
    # 4. Render
    set_blend_mode(SRC_ALPHA, ONE_MINUS_SRC_ALPHA)
    set_depth_write(False)
    bind_texture_stack(textures[best_axis])
    
    for poly in polygons:
        # Vertex shader computes texcoords as affine map of vertexMC [S2]
        # Fragment shader samples 2D texture, multiplies luminance by tissue color [S3]
        draw_polygon(poly)

def dot(v1, v2):
    return sum(a * b for a, b in zip(v1, v2))

def normalize(v):
    mag = dot(v, v) ** 0.5
    return [x / mag for x in v]

def argmin(lst):
    return lst.index(min(lst))

def generate_fan_polygon(plane_pos, volume_box):
    # Uses 256-case plane/cube clipping table to intersect plane with box [S2]
    # Returns a triangle-fan geometry with 3D texture coordinates
    return {"vertices": [], "texcoords": [], "order": "back-to-front"}

def set_blend_mode(src, dst):
    # Configures hardware blending state
    pass

def set_depth_write(enabled):
    # Disables depth buffer writes
    pass

def bind_texture_stack(stack):
    # Binds the selected axis texture set to GPU
    pass

def draw_polygon(poly):
    # Rasterizes polygon, invoking vertex/fragment shaders
    pass
```

**Edge Cases and Pitfalls:**
*   **Bilinear Interpolation Artifacts:** Since only 2D textures are used, interpolation is bilinear [S1]. This ignores the third dimension, causing visible "banding" artifacts around the volume [S3].
*   **Popping Artifacts:** Switching between X, Y, and Z texture sets during rotation causes visible jumps due to changes in interpolation artifacts or memory swap times [S3].
*   **Sampling Frequency Change:** Rotating the camera changes the sampling frequency [S3]. If planes become perpendicular to the projection plane, they may disappear from the final image [S3].
*   **VTK OIT Conflict:** VTK 9's default Order-Independent Translucency (OIT) breaks compositing by averaging colors (X-ray look) [S2]. Must disable OIT (`UseOITOff`) to use legacy blending [S2].
*   **Off-Origin Volume Bug:** If the volume box is offset from the origin, VTK's AUTO VBO shift-scale re-centers vertex data [S2]. Deriving texcoords from `vertexMC` then reads the wrong corner (density 0) [S2]. Fix: Set `DISABLE_SHIFT_SCALE` [S2].
*   **GLES3 Precision:** On WebGL2/GLES3, `sampler3D` has no default precision (compile error if unqualified) and `sampler2D` defaults to `lowp` (insufficient for LUT addressing) [S2]. Both must be declared `highp` [S2].

## Subtopics

### [Novel Sampling Scheme for Slice Based Volume Rendering](953-novel-sampling-scheme-for-slice-based-volume-rendering.md)
A variant that eliminates stack-switching and jagged boundary artifacts by generating proxy geometry using multiple object-aligned 2D texture stacks clipped by planes extending from the front-facing edges of the volume's bounding box.

## Operating characteristics
*   **Time Complexity:** Linear with respect to the number of slices $N$ and the number of pixels covered by the proxy geometry. $N$ is determined by the formula $N = 2 \cdot (10 + \text{max\_planes} \cdot q^3)$ [S2].
*   **Memory Complexity:** $O(P^2 \cdot (D_x + D_y + D_z))$ for storing 2D texture stacks, where $P$ is the power-of-two padding dimension [S3]. For a $181 \times 217 \times 181$ volume, this requires 579 textures of $256 \times 256$ pixels, totaling ~37.9 MB [S3].
*   **Performance:** Very fast on modern GPUs due to reliance on rasterization hardware and hardware blending [S2, S3]. Historically enabled real-time rendering on early GPUs (e.g., GeForce 256) [S1].
*   **Strengths:** Intuitive, easy to implement, leverages mature 2D texture hardware, efficient for interactivity [S1, S3].
*   **Limitations:** Severe artifacts from bilinear interpolation (banding), popping artifacts during axis switching, reduced accuracy compared to trilinear interpolation [S1, S3].
*   **When to Use:** Legacy hardware, simple interactive visualization where speed is prioritized over high-fidelity volume quality [S1, S3].
*   **When Not to Use:** High-quality offline rendering, scenes requiring precise trilinear interpolation, or when popping artifacts are unacceptable [S1, S3].

## Background
*   **Origin:** Developed after 2D texture mapping became available on GPUs, preceding 3D texture support. Engel et al. provided an excellent introduction to these algorithms [S1].
*   **History:** Studied extensively in the early 2000s (e.g., VolumeRover2 by Thane Bajaj, UT Austin, 2002-2003) [S2]. Rezk-Salama et al. later proposed multi-texture methods to approximate trilinear interpolation using bilinear operations [S1].
*   **Usage:** Used in early real-time volume renderers and educational demonstrations. Ported to modern contexts like `cvc::volslice` for compatibility with legacy rendering pipelines [S2].

## Sources
[S1] A system for real-time rendering of compressed time-varying volume data, https://doi.org/10.7939/r3041m, fetched 2026-10-26
[S2] cvc::volslice: port VolumeRover2's view-aligned slice volume renderer, https://github.com/transfix/libcvc/pull/332, fetched 2026-10-26 (unofficial)
[S3] Texture Mapping Based Volume Rendering, https://www.ciroduran.com/files/Tesis.pdf, fetched 2026-10-29
