# Two-pass volume rendering

Also called: Light/Volume two-pass, Slice-based two-pass volume rendering

## Overview
Two-pass volume rendering is a direct volume ray casting technique that separates light attenuation computation from color compositing into two distinct rasterization passes [S1, unofficial]. It differs from single-pass methods by decoupling these operations into a two-stage data dependency between a dedicated light buffer and the frame buffer [S1, unofficial].

## How it works
The method generates proxy geometry (slice planes) along the view direction and processes them in two stages per slice [S1, unofficial]. The first stage (Light Pass) accumulates light attenuation [S1, unofficial]. The second stage (Eye Pass) composites color and opacity using the attenuated light from the first stage [S1, unofficial].

### Inputs and Outputs
*   **Input**: 3D volume texture (density/scalar field), 1D or 2D transfer function texture, camera matrices (eye and light view-projection), slice geometry, number of slices [S1, unofficial].
*   **Output**: Final color buffer (framebuffer) [S1, unofficial].
*   **Intermediate**: Light buffer (2D texture storing accumulated light color and opacity) [S1, unofficial].

### Data Structures

**Push Constants**
The implementation uses two distinct push constant structures passed to vertex and fragment shaders [S1, unofficial].

1.  `LightPC` (Light Pass):
    *   `lightMVP`: `mat4` (Light view-projection matrix).
    *   `opacityCorrection`: `float` (Scalar multiplier for opacity).
    *   `m_densityMin`: `float` (Lower density threshold).
    *   `m_densityMax`: `float` (Upper density threshold).
    *   Total size: `sizeof(LightPC)` [S1, unofficial].

2.  `EyePC` (Eye Pass):
    *   `eyeMVP`: `mat4` (Eye view-projection matrix).
    *   `lightMVP`: `mat4` (Light view-projection matrix, reused for sampling light buffer).
    *   `opacityCorrection`: `float`.
    *   `m_ambient`: `float` (Ambient light intensity).
    *   `m_densityMin`: `float`.
    *   `m_densityMax`: `float`.
    *   Total size: `sizeof(EyePC)` [S1, unofficial].

**Descriptor Set Bindings**
*   **Volume Set**:
    *   Binding 0: Volume texture (3D) [S1, unofficial].
    *   Binding 1: Transfer function texture (1D or 2D) [S1, unofficial].
*   **Eye Set**:
    *   Binding 0: Volume texture (3D) [S1, unofficial].
    *   Binding 1: Transfer function texture (1D or 2D) [S1, unofficial].
    *   Binding 2: Light buffer texture (2D) [S1, unofficial].
*   **Composite Set**:
    *   Binding 0: Eye buffer (Frame buffer) [S1, unofficial].

**Light Buffer Layout**
*   Resolution: Not given in the sources.
*   Format: Typically RGBA8, storing accumulated light color and opacity [S1, unofficial].
*   Initialization: Cleared to light color with alpha set to $0.0$ [S1, unofficial].

### Numbered Steps

1.  **Generate Slice Geometry**:
    *   Calculate view direction from camera [S1, unofficial].
    *   Generate slice proxy geometry spanning volume bounds [S1, unofficial].
    *   Order slices back-to-front relative to the light source. The implementation sets `reverse=true` to generate geometry ordered back-to-front for the "over" operator [S1, unofficial].

2.  **Initialize Buffers**:
    *   Clear Light Buffer: Set color to light color, alpha to $0.0$ [S1, unofficial].
    *   Clear Eye Buffer: Set to black $(0,0,0,0)$ [S1, unofficial].

3.  **Iterate Slices**:
    For each slice $i$ from light end to eye end:
    a.  **Light Pass**:
        *   Bind Light Buffer as render target [S1, unofficial].
        *   Bind Volume Set descriptors [S1, unofficial].
        *   Push `LightPC` constants [S1, unofficial].
        *   Draw slice geometry [S1, unofficial].
        *   Blend Mode: Premultiplied Over [S1, unofficial].
        *   *Shader Logic*: Sample volume, compute density, accumulate light attenuation [S1, unofficial]. Write to light buffer.
    b.  **Eye Pass**:
        *   Bind Eye Buffer as render target [S1, unofficial].
        *   Bind Eye Set descriptors (including Light Buffer at binding 2) [S1, unofficial].
        *   Push `EyePC` constants [S1, unofficial].
        *   Draw slice geometry [S1, unofficial].
        *   Blend Mode: 'Under' (accumulates color and opacity front-to-back) [S1, unofficial].
        *   *Shader Logic*: Sample volume, apply transfer function, sample light buffer for attenuation, composite color [S1, unofficial].

4.  **Final Composite**:
    *   The Eye Buffer now contains the final composited image [S1, unofficial].

### Formulas

**Opacity Correction**
To maintain correct total opacity regardless of the number of slices rendered, an opacity correction scalar is applied [S1, unofficial]:

$$
opacityCorrection = \frac{kReferenceSlices}{m\_numSlices}
$$

Where:
*   $kReferenceSlices$ is a reference slice count (e.g., 256) [S1, unofficial].
*   $m\_numSlices$ is the actual number of slices rendered [S1, unofficial].

**Light Attenuation (Light Pass)**
The light buffer accumulates light. For a slice with opacity $\alpha$ and transmittance $T$:
$$
T_{new} = T_{old} \cdot (1 - \alpha)
$$
$$
L_{new} = L_{old} + L_{incident} \cdot T_{old} \cdot \alpha
$$
*(Note: Specific shader implementation details for light accumulation are not fully provided in the source, but the 'under' blend in the eye pass implies standard front-to-back compositing using the pre-attenuated light.)* [S1, unofficial]

**Color Compositing (Eye Pass)**
Using 'Under' blending, the eye pass composites the slice color $C_s$ and opacity $\alpha_s$ with the existing eye buffer color $C_e$ and opacity $\alpha_e$:
$$
C_{new} = C_s \cdot \alpha_s + C_e \cdot \alpha_e \cdot (1 - \alpha_s)
$$
$$
\alpha_{new} = \alpha_s + \alpha_e \cdot (1 - \alpha_s)
$$
*(Note: The 'Under' blend mode in hardware typically performs this accumulation. The light buffer provides the attenuation factor for the slice color.)* [S1, unofficial]

### Parameters

*   `opacityCorrection`: Scalar multiplier applied to slice opacity. Calculated as the ratio of a reference slice count to the actual slice count [S1, unofficial].
*   `m_densityMin`, `m_densityMax`: Density thresholds passed to shaders [S1, unofficial].
*   `kLightSize`: Not given in the sources.
*   `kReferenceSlices`: Reference slice count for opacity correction (e.g., 256) [S1, unofficial].
*   `m_numSlices`: Number of slices to render (e.g., 16 to 768) [S1, unofficial].
*   `m_densityScale`: Scale factor for density values (e.g., 0.0 to 4.0) [S1, unofficial].

### Edge Cases and Pitfalls

*   **Upside Down Volume**: Volume data uploaded from CPU stores rows top-to-bottom, causing the rendered volume to appear upside down. This is handled by a `flipY` function that mirrors the data along the Y axis before GPU upload [S1, unofficial].
*   **Slice Iteration Order**: Slice iteration order must strictly match the front-to-back requirement from the light. The implementation fixes `reverse=true` unconditionally, as no modern GPU configuration requires the opposite order [S1, unofficial].
*   **Opacity Consistency**: Without `opacityCorrection`, changing the number of slices changes the total opacity and image intensity. The correction factor ensures image intensity stays constant as slice count changes [S1, unofficial].

### Pseudocode

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
# Constants
kReferenceSlices = 256

# Initialization
def create_texture_2d(width, height, format):
    return Texture(width, height, format)

def clear_render_target(target, color, alpha):
    target.clear(color=color, alpha=alpha)

def generate_slices(box_half, view_dir, num_slices, reverse):
    return SliceProxyGeometry.generate(box_half, box_half, view_dir, num_slices, reverse)

def bind_descriptor_set(set_type):
    # Binds the appropriate descriptor set (Volume, Eye, or Composite)
    pass

def push_constants(constants):
    # Uploads push constant data to the shader pipeline
    pass

def draw_slice(slice_geometry):
    # Rasterizes the slice geometry
    pass

def set_blend_mode(mode):
    # Sets the GPU blend mode (e.g., PREMULTIPLIED_OVER, UNDER)
    pass

# Setup
light_buffer = create_texture_2d(1024, 1024, "RGBA8")
eye_buffer = create_texture_2d(framebuffer_width, framebuffer_height, "RGBA8")
volume_tex = load_volume_3d(volume_data, flip_y=True)
tf_tex = load_transfer_function(tf_data)

# Clear buffers
clear_render_target(light_buffer, color=light_color, alpha=0.0)
clear_render_target(eye_buffer, color=(0, 0, 0, 0), alpha=0.0)

# Generate slice geometry
view_dir = camera.forward()
slices = generate_slices(
    box_half=m_boxHalf,
    view_dir=view_dir,
    num_slices=m_numSlices,
    reverse=True  # Generates back-to-front for the "over" operator
)

# Render Loop
for slice in slices:
    # --- Light Pass ---
    bind_render_target(light_buffer)
    bind_descriptor_set("Volume") # Binding 0: Volume, Binding 1: TF
    push_constants(LightPC(
        lightMVP=light_view_proj,
        opacityCorrection=kReferenceSlices / m_numSlices,
        m_densityMin=m_densityMin,
        m_densityMax=m_densityMax
    ))
    draw_slice(slice)
    set_blend_mode("PREMULTIPLIED_OVER")

    # --- Eye Pass ---
    bind_render_target(eye_buffer)
    bind_descriptor_set("Eye") # Binding 0: Volume, Binding 1: TF, Binding 2: Light Buffer
    push_constants(EyePC(
        eyeMVP=eye_view_proj,
        lightMVP=light_view_proj,
        opacityCorrection=kReferenceSlices / m_numSlices,
        m_ambient=m_ambient,
        m_densityMin=m_densityMin,
        m_densityMax=m_densityMax
    ))
    draw_slice(slice)
    set_blend_mode("UNDER") # Front-to-back compositing

# Final image is in eye_buffer
```

## Subtopics

### [Two-Pass Lighting and Shading](13390-two-pass-lighting-and-shading.md)
Decouples light transport (attenuation/scattering) from final compositing. Pre-computes light accumulation in a voxel grid for real-time rendering. Differs from standard ray casting by moving light integration off the critical path.

### [Half-angle slicing](1255-half-angle-slicing.md)
Uses half-angle vectors for slice generation to optimize ray traversal.

### [Two-Pass Ray Marching Volume Rendering](5965-two-pass-ray-marching-volume-rendering.md)
Uses ray marching in two passes, potentially for adaptive resolution or different sampling strategies.

### [Multi-pass GPU Ray-Casting with Early Ray Termination](7009-multi-pass-gpu-ray-casting-with-early-ray-termination.md)
Incorporates early ray termination across multiple passes to optimize performance.

### [Locally Adaptive Volume Rendering](13045-locally-adaptive-volume-rendering.md)
Adapts rendering resolution or sampling density based on local volume features.

## Operating characteristics
*   **Time Complexity**: $O(N \cdot S)$, where $N$ is the number of slices and $S$ is the number of samples per slice (or slice area). Two passes per slice doubles the constant factor compared to single-pass, but allows optimization of light calculation [G].
*   **Memory Complexity**: $O(kLightSize^2)$ for the light buffer, plus standard volume and transfer function storage [G].
*   **Performance**: Depends heavily on GPU blending hardware support for 'Under' and 'Over' modes. Efficient reuse of light buffer avoids recomputing attenuation [G].
*   **Strengths**: Decouples complex light transport from compositing, allowing for more sophisticated lighting models without impacting compositing performance. Leverages hardware blending [G].
*   **Limitations**: Requires two render passes per slice, increasing draw call overhead. Light buffer resolution must be sufficient to avoid artifacts [G].
*   **When to Use**: When complex lighting is required and can be pre-computed or accumulated efficiently. When hardware blending is available and efficient [G].
*   **When Not to Use**: When memory bandwidth is a severe bottleneck (light buffer adds pressure). When simple lighting suffices (single-pass may be more efficient) [G].

## Background
The technique is based on the design described in GPUGems Fig. 39-1 [S1, unofficial]. It is implemented in the GPUGemsVolumeRendering demo [S1, unofficial]. The approach builds on the core loop structure of image-order rendering and direct volume ray casting, splitting the computational engine across two passes to persist intermediate state in 2D textures [G].

## Sources
[S1] sho3la/GPUGemsVolumeRendering, https://github.com/sho3la/GPUGemsVolumeRendering, fetched 2026-10-26 (unofficial)
