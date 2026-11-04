# Volume Raycasting on GPU

Also called: GPU-Based Raycasting, GPU-accelerated raycasting, Multi-pass Volume Raycasting, Stream model raycasting, fragment program raycasting

## Overview
Volume Raycasting on GPU is a direct volume rendering (DVR) strategy that exploits the programmable parallelism of modern graphics processing units (GPUs) to generate and trace a single ray per screen pixel through a 3D volume dataset [S2]. Unlike fixed-pipeline Texture Mapping (TM) methods, which apply shading to every fragment regardless of contribution and suffer from striping artifacts under perspective projection due to uniform sampling steps, GPU raycasting encodes the rendering pipeline in fragment shaders, enabling adaptive sampling, early ray termination, and empty space skipping [S2]. It builds on [Ray Marching and Discretization](2788-ray-marching-and-discretization.md) by implementing the ray traversal and compositing logic directly within the GPU's fragment processing stage [S2].

## How it works
The method typically employs a multi-pass algorithm where initial passes precompute ray geometry, and subsequent passes perform parallel raymarching for each pixel [S1, unofficial]. The core implementation involves storing the volume dataset as a 3D texture mapped to a colored unit cube bounding box [S2].

**Inputs and Outputs:**
*   **Input:** A 3D texture containing the volume scalar field; camera parameters (origin, view direction, projection); transfer function (TF) mapping scalar values to color and opacity; and a preset opacity threshold for early termination [S2].
*   **Output:** A 2D image where each pixel contains the accumulated color and opacity of the ray traversing the volume [S2].

**Data Structures:**
1.  **3D Texture:** Stores the volume data. The bounding box is treated as a unit cube. The texture coordinates at the eight corners of this cube are encoded in the RGB color channels of a 2D texture representing the cube's faces [S2].
2.  **Direction Vector Texture (2D):** Stores the normalized direction vectors for the casting ray for each pixel. This is derived by subtracting the color values of the back face of the bounding cube from the front face and normalizing the result [S2].
3.  **Intermediate Framebuffer/Texture:** Used in multi-pass implementations to store accumulated color and opacity between raymarching steps, allowing for early termination checks [S1, unofficial].

**Algorithm Steps:**

1.  **Ray Direction and Origin Computation:**
    *   Render the front face of the colored unit cube bounding box into an intermediate texture [S2].
    *   Render the back face of the colored unit cube bounding box into another intermediate texture [S2].
    *   For each pixel, subtract the RGB color values of the back face from the front face. Normalize the resulting vector to obtain the ray direction $v$ [S2].
    *   The ray origin $O$ is defined as the pixel position on the front face of the bounding cube [S2].
    *   Store the normalized direction vectors in a 2D texture `DIR[x][y]` [S1, unofficial].

2.  **Raymarching Loop (Fragment Shader):**
    *   For each pixel, retrieve the ray origin $O$ and direction $v$ (or `DIR[x][y]`) [S2].
    *   Initialize accumulated color $C_{acc} = (0,0,0,0)$ and accumulated opacity $\alpha_{acc} = 0$ [G].
    *   Define a parametric ray equation: $r(t) = O + v \times t$, where $t$ is the distance along the ray [S2]. Alternatively, using the texture coordinate formulation: $r = P + d \cdot DIR[x][y]$, where $P$ is the per-vertex position of the front face, $d$ is the total raymarched distance, and $r$ represents the 3D texture coordinates used to sample voxel data [S1, unofficial].
    *   Uniformly sample the volume along the ray. In a single-pass implementation, this is done using GPU Shader Model 3.0 loop capabilities [S2]. In a multi-pass implementation, the full raymarch is split into chunks (e.g., $M$ steps per pass) [S1, unofficial].
    *   At each sampling point $r$:
        *   Sample the 3D texture to get the scalar value $s$ [G].
        *   Apply the Transfer Function (TF) to map $s$ to color $c$ and opacity $\alpha$ [S2].
        *   Perform front-to-back blending:
            $$ C_{acc} = C_{acc} + c \cdot \alpha \cdot (1 - \alpha_{acc}) $$
            $$ \alpha_{acc} = \alpha_{acc} + \alpha \cdot (1 - \alpha_{acc}) $$
    *   **Termination Check:** Terminate the ray if:
        *   The ray leaves the volume bounding box [S2].
        *   The accumulated opacity $\alpha_{acc}$ exceeds a preset threshold [S2].
        *   (In multi-pass) The accumulated opacity in the intermediate texture exceeds the threshold [S1, unofficial].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
# Fragment Shader Implementation for a Single Ray per Pixel
# Inputs:
#   DIR[xy]: 2D texture storing normalized ray direction vectors
#   VOL: 3D texture storing the volume data
#   TF: Function mapping scalar value to (color, opacity)
#   THRESHOLD: Preset opacity threshold for early termination
#   FRONT_FACE_POS: Per-vertex position of the front face of the bounding box (interpolated)
#   STEP_SIZE: Constant uniform step size for sampling

def fragment_shader(pixel_coords):
    # 1. Get Ray Geometry
    # Ray direction from the precomputed texture
    v = DIR[pixel_coords.x][pixel_coords.y]
    
    # Ray origin is the interpolated position of the front face of the bounding box
    O = FRONT_FACE_POS
    
    # 2. Initialize Accumulators
    acc_color = (0.0, 0.0, 0.0, 0.0)
    acc_opacity = 0.0
    
    # 3. Raymarching Loop
    # Determine total steps or use a loop with early exit
    # Note: In a real GPU shader, this might be a fixed loop or split across passes
    t = 0.0
    while t < MAX_DISTANCE:
        # Calculate current 3D texture coordinate
        # r = O + v * t
        r = O + v * t
        
        # Check if r is within the volume bounds [0,1]^3
        if not is_inside_volume(r):
            break
            
        # Sample volume data
        scalar_value = sample_3d_texture(VOL, r)
        
        # Map to color and opacity
        color, opacity = TF(scalar_value)
        
        # Front-to-back blending
        # C_new = C_old + c * alpha * (1 - alpha_old)
        # alpha_new = alpha_old + alpha * (1 - alpha_old)
        
        contribution = color * opacity * (1.0 - acc_opacity)
        acc_color = acc_color + contribution
        acc_opacity = acc_opacity + opacity * (1.0 - acc_opacity)
        
        # Early termination check
        if acc_opacity >= THRESHOLD:
            break
            
        # Advance along ray
        t = t + STEP_SIZE
        
    return acc_color
```

**Edge Cases and Pitfalls:**
*   **Perspective Projection:** Fixed-pipeline TM methods suffer from striping artifacts because uniform sampling steps in screen space do not correspond to uniform steps in world space under perspective projection. GPU raycasting handles this via adaptive sampling steps or by computing ray directions per pixel, maintaining uniformity in world space [S2].
*   **Memory Bandwidth:** Multi-pass implementations require storing intermediate results in temporary framebuffers, which can suffer from low efficiency due to bus bandwidth and latency between CPU and GPU [S2].
*   **Precision:** The encoding of texture coordinates in RGB channels for direction computation requires careful normalization to avoid precision errors in the direction vector [S2].

## Subtopics

### Single-pass Raycasting
A raycasting framework that avoids multiple rendering passes (casting rays and storing intermediate results in temporary framebuffers) to address the low efficiency of multi-pass pipelines on current graphics hardware architectures [S2]. Compared with multi-pass rendering, single-pass solutions offer advantages in efficiency, accuracy, and flexibility [S2]. It has been extended by Stegmaier and colleagues for spectral DVR using the Kubelka–Munk theory and by Hadwiger et al. to display implicit isosurfaces from volumes on rectilinear 3D grids using hierarchical grid representations for empty space skipping [S2].

## Operating characteristics
*   **Time Complexity:** Linear with respect to the number of sampling steps per ray and the number of pixels. Adaptive sampling and early termination reduce the average number of steps per ray [S2].
*   **Memory Complexity:** Dominated by the storage of the 3D volume texture and the intermediate framebuffers (in multi-pass implementations) [S2].
*   **Measured Performance:** Used for real-time display of fused multimodal brain datasets for neurosurgical planning and other medical imaging applications [S2]. Performance is limited by GPU memory bandwidth and the cost of 3D texture sampling [S2].
*   **Strengths:** Flexible pipeline allowing for acceleration techniques like early ray termination and empty space skipping; fragment-based shading applies computation only to contributing fragments; adaptive sampling improves image quality under perspective projection [S2].
*   **Limitations:** Multi-pass versions suffer from framebuffer read/write overhead; single-pass versions may have limited flexibility for complex acceleration structures compared to CPU-based methods [S2].
*   **When to use:** Real-time visualization of large 3D datasets, particularly in medical imaging where interactive frame rates are required [S2].
*   **When not to use:** When the volume data exceeds available GPU texture memory and efficient out-of-core streaming is not implemented; when complex light transport effects beyond simple absorption and emission are required without significant shader complexity [S2].

## Background
The general strategy of exploiting programmable GPUs for raycasting was proposed by Krüger and Westermann, and Purcell et al., who described a stream model exploiting the intrinsic parallelism and efficient communication on modern GPUs [S2]. The original GPU-based solutions were based on multiple rendering passes [S2]. Later, Stegmaier and colleagues developed single-pass frameworks to improve efficiency [S2]. The method is widely used in medical imaging, including neurosurgical planning, thoracic CT segmentation, coronary artery disease diagnosis, virtual autopsy, lung cancer screening, virtual endoscopy, and 4D cardiac dataset visualization [S2].

## Sources
[S1] PDF CS 519: Scientific Visualization - University of Illinois Urbana-Champaign, https://courses.grainger.illinois.edu/cs519/fa2017/Lecture%2012%20-%20Volume%20Visualization.pdf, fetched 2026-10-27 (unofficial)
[S2] Volume Visualization: A Technical Overview with a Focus on ..., https://pmc.ncbi.nlm.nih.gov/articles/PMC3138940/, fetched 2026-10-27
