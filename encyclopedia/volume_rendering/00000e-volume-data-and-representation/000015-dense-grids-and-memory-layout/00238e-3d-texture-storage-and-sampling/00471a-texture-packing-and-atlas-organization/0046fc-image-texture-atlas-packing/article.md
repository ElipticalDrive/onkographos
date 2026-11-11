# Image Texture Atlas Packing

Also called: 2D Texture Atlas Emulation, ImageTextureAtlas, WebGL 1.0 3D Texture Emulation, X3DTexture2DNode

## Overview
Image Texture Atlas Packing is a technique for representing 3D volumetric data (such as MRI or CT scans) as a set of 2D image slices arranged into a single 2D texture atlas. This method solves the problem of rendering 3D volumes on hardware that lacks native 3D texture support, specifically WebGL 1.0, by emulating 3D sampling through 2D texture lookups and fragment shader interpolation [S1]. It differs from [3D Volume Atlas Packing](7158-3d-volume-atlas-packing.md) by explicitly targeting the constraints of legacy web graphics APIs using a matrix-based 2D layout, and from [Texture Uniting (Multi-Channel Packing)](2470-texture-uniting-multi-channel-packing.md) by organizing spatial slices rather than semantic channels into a single spatial domain [S1].

## How it works
**Inputs and Outputs:**
*   **Input:** A 3D volume dataset defined by dimensions $(D_x, D_y, D_z)$, where $D_z$ is the number of 2D slices. Optional inputs include surface normals (gradient vectors) or transfer function data.
*   **Output:** A single 2D texture (e.g., PNG, Canvas) containing all slices arranged in a matrix, along with metadata defining the layout (slices per axis, channel usage, sort order).

**Data Structures:**
The method relies on the `ImageTextureAtlas` node definition (based on X3D standards) with the following fields [S1]:
*   `numberOfSlices` (SFInt32): Total number of slices $D_z$.
*   `slicesOverX` (SFInt32): Number of slices tiled along the texture width (S-axis).
*   `slicesOverY` (SFInt32): Number of slices tiled along the texture height (T-axis).
*   `channels` (SFString): Specifies which color channels store data. Default is `"R"`. Can include `"G"`, `"B"`, `"A"` to store larger datasets or gradient vectors.
*   `sortOrder` (SFString): Defines tiling order. Values are `"ROW"` (default) or `"CHANNEL"`.
*   `containerField` (SFString): Targets the input field of the parent node (e.g., `voxels` for volume data, `surfaceNormals` for gradients).

**Layout and Mapping:**
The 3D volume is flattened into a 2D texture. The mapping from logical 3D coordinates $(u, v, w)$ to physical 2D texture coordinates $(s, t)$ depends on the `sortOrder` and `channels` configuration [S1].

1.  **ROW Order (Default):** Slices are tiled contiguously in rows. If `slicesOverX` is $N_x$, the slice index $w$ maps to texture coordinates:
    $$ s = \frac{u + (w \pmod{N_x}) \cdot W_{slice}}{W_{atlas}} $$
    $$ t = \frac{v + \lfloor w / N_x \rfloor \cdot H_{slice}}{H_{atlas}} $$
    Where $W_{slice}, H_{slice}$ are the dimensions of a single slice, and $W_{atlas}, H_{atlas}$ are the atlas dimensions.

2.  **CHANNEL Order:** Slices are stored in contiguous color channels. This is used when `channels` includes multiple characters (e.g., `"RGBA"`). The slice index $w$ maps to the channel index $c = w \pmod{4}$ (assuming RGBA packing) and the spatial coordinates remain within the slice. The shader must sample the specific channel based on $w$.

**Pseudocode:**
The following Python-like pseudocode demonstrates the coordinate reconstruction in a fragment shader context. It assumes the texture is bound as a 2D texture and the shader receives the logical 3D ray intersection point.

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def reconstruct_texture_coordinate(logical_u, logical_v, logical_w, 
                                   atlas_width, atlas_height, 
                                   slice_width, slice_height,
                                   slices_over_x, channels, sort_order):
    """
    Reconstructs 2D texture coordinates (s, t) and channel selection
    from logical 3D volume coordinates.
    
    Args:
        logical_u, logical_v, logical_w: Normalized coordinates [0, 1] in volume space.
        atlas_width, atlas_height: Dimensions of the 2D texture atlas.
        slice_width, slice_height: Dimensions of a single volume slice.
        slices_over_x: Number of slices per row in the atlas.
        channels: String indicating used channels (e.g., "R", "RGBA").
        sort_order: "ROW" or "CHANNEL".
        
    Returns:
        (s, t, channel_index): Texture coordinates and channel index (0-3).
    """
    # Determine total slices per channel based on channel string length
    num_channels = len(channels)
    
    # Calculate slice index in the logical volume
    # Note: logical_w is normalized [0,1]. Convert to integer slice index.
    # Assuming volume depth D_z is known or derived from atlas config.
    # For this implementation, we assume logical_w maps directly to slice index
    # relative to the total number of slices packed.
    
    # If using CHANNEL sort order, the 'w' coordinate might span across channels
    # If using ROW sort order, 'w' spans within the 2D grid.
    
    if sort_order == "ROW":
        # Map logical_w to 2D grid position
        # Total slices = slices_over_x * slices_over_y (implicitly)
        # We need the integer slice index. 
        # Let's assume we pass total_slices or derive it. 
        # For generality, we calculate row/col from w.
        
        # Normalize w to [0, total_slices)
        # In a real shader, total_slices is a uniform.
        # Here we simulate the mapping logic.
        
        # Calculate which slice index this w corresponds to.
        # This requires knowing the total number of slices.
        # Let's assume total_slices is provided or calculated from atlas/slice dims.
        # total_slices = (atlas_width / slice_width) * (atlas_height / slice_height)
        
        # For the purpose of the algorithm, we map w to a 1D index.
        # w_norm = logical_w
        # slice_idx = int(w_norm * total_slices)
        
        # Row and Col in the atlas grid
        col = slice_idx % slices_over_x
        row = slice_idx // slices_over_x
        
        # Calculate s, t
        s = (logical_u * slice_width + col * slice_width) / atlas_width
        t = (logical_v * slice_height + row * slice_height) / atlas_height
        
        # Channel is always 0 (Red) for standard ROW unless multi-channel logic is applied
        # But ROW usually implies spatial tiling. If channels are used in ROW, 
        # it's more complex. The source says ROW tiles Z-axis slices.
        # If channels > 1, ROW might still apply to the spatial grid, 
        # but the source links CHANNEL order to multi-channel storage.
        
        channel_idx = 0 # Default for ROW if not specified otherwise
        
    elif sort_order == "CHANNEL":
        # Slices are stored in contiguous channels.
        # logical_w determines the channel index.
        # w_norm = logical_w
        # channel_idx = int(w_norm * num_channels)
        # channel_idx = clamp(channel_idx, 0, num_channels - 1)
        
        # Spatial coordinates (u, v) map directly to the slice in that channel.
        s = logical_u
        t = logical_v
        
    else:
        raise ValueError("Unknown sort order")

    return (s, t, channel_idx)

def sample_volume(logical_u, logical_v, logical_w, texture_sampler, 
                  atlas_width, atlas_height, slice_width, slice_height,
                  slices_over_x, channels, sort_order):
    """
    Samples the volume data using the reconstructed coordinates.
    """
    s, t, channel_idx = reconstruct_texture_coordinate(
        logical_u, logical_v, logical_w,
        atlas_width, atlas_height, slice_width, slice_height,
        slices_over_x, channels, sort_order
    )
    
    # Sample the 2D texture
    # In GLSL, this would be texture2D(tex, vec2(s, t))
    pixel = texture2D(texture_sampler, vec2(s, t))
    
    # Extract the specific channel
    channel_values = [pixel.r, pixel.g, pixel.b, pixel.a]
    value = channel_values[channel_idx]
    
    return value
```

**Parameters:**
*   `slicesOverX`, `slicesOverY`: Must satisfy `slicesOverX * slicesOverY >= numberOfSlices`. Default 0 implies auto-calculation or error if not provided [S1].
*   `channels`: Default `"R"`. Use `"RGBA"` to store up to 4 times the data volume or gradient vectors (RGB) [S1].
*   `sortOrder`: Default `"ROW"`. Use `"CHANNEL"` for multi-channel packing [S1].

**Edge Cases and Pitfalls:**
*   **GPU Texture Limits:** WebGL 1.0 has a maximum 2D texture size (e.g., 4096x4096). Large volumes must be split or use multi-channel packing to fit [S1].
*   **Precision Loss:** Video encoding for 4D rendering introduces accuracy loss [S1].
*   **Interpolation:** Trilinear interpolation is simulated in the fragment shader by sampling the 2D atlas. This requires careful handling of boundary conditions and slice edges [S1].

## Subtopics

### 4D Volume Rendering
Extends the method by creating an atlas for each time step, encoding them into a video loop, and updating the texture in real-time via the canvas API. It uses a hidden HTML5 `<video>` element to drive the texture updates [S1].

### Surface Normal Packing
Encodes 3-component gradient vectors (surface normals) into the RGB channels of the atlas (R:X, G:Y, B:Z). This allows normal-based shading without 3D textures [S1].

### Dynamic Transfer Functions
Uses the 2D canvas API to modify the transfer function texture directly, enabling native HTML5 editors for color/opacity mapping [S1].

## Operating characteristics
*   **Time Complexity:** Sampling is $O(1)$ per pixel, but requires additional arithmetic in the shader to map 3D coordinates to 2D atlas coordinates.
*   **Memory Complexity:** $O(D_x \cdot D_y \cdot D_z)$ for the volume data, but stored in a single 2D texture object. Multi-channel packing increases effective capacity by a factor of up to 4 (RGBA) [S1].
*   **Performance:** Dependent on GPU 2D texture sampling speed. Less memory intensive than full 3D texture storage on hardware lacking 3D texture support. MPR (Multi-Planar Reconstruction) using this method is less memory expensive than full volume rendering as it only resamples data near the defined plane [S1].
*   **Strengths:** Enables 3D volume rendering on WebGL 1.0 hardware. Supports dynamic updates via Canvas API.
*   **Limitations:** Limited by 2D texture size. Potential precision loss in 4D video-based updates. Requires custom shader logic for coordinate mapping [S1].

## Background
The method was proposed to enable interactive visualization of volumetric data in Web-based GPU accelerated volume rendering algorithms, specifically within the X3DOM framework. It addresses the limitation of WebGL 1.0, which does not support the Texturing3D component required for native 3D textures [S1]. The approach builds on earlier work by Congote et al. (2011) and was integrated into the X3D Volume Rendering Component to support ubiquitous deployment on web platforms [S1]. It is used in X3DOM for web-based volume rendering [S1].

## Sources
[S1] Community-driven extensions to the X3D volume rendering component, https://doi.org/10.1145/3055624.3075945, fetched 2026-11-02
