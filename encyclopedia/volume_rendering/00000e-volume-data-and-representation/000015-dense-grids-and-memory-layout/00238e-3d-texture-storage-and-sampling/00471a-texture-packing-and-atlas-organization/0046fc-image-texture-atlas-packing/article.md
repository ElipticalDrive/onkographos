# Image Texture Atlas Packing

Also called: 2D Texture Atlas Emulation, ImageTextureAtlas, WebGL 1.0 3D Texture Emulation, X3DTexture2DNode

## Overview
Image Texture Atlas Packing is a technique for representing 3D volumetric data (e.g., MRI/CT scans) as a set of 2D image slices arranged into a single 2D texture (atlas) to emulate 3D textures in environments lacking native 3D texture support, such as WebGL 1.0 [S1]. It differs from [3D Volume Atlas Packing](7158-3d-volume-atlas-packing.md) by explicitly targeting the WebGL 1.0 constraint where `Texturing3D` is unsupported, forcing a 2D matrix layout rather than a true 3D grid, and from [Texture Uniting (Multi-Channel Packing)](2470-texture-uniting-multi-channel-packing.md) by organizing discrete spatial slices into a 2D grid rather than packing semantic channels into RGBA components of a single volume [S1].

## How it works
**Inputs and Outputs:**
*   **Input:** A sequence of 2D image slices representing the Z-axis of a volume, or gradient data for surface normals.
*   **Output:** A single 2D texture resource (e.g., PNG, Canvas) containing the packed atlas, and a shader-compatible coordinate mapping for 3D sampling.

**Data Structures:**
The method relies on the `ImageTextureAtlas` node definition within the X3D specification. The layout is defined by the following fields [S1]:
*   `numberOfSlices` (SFInt32): Total number of 2D slices in the volume.
*   `slicesOverX` (SFInt32): Number of slices along the X-axis of the 2D atlas grid.
*   `slicesOverY` (SFInt32): Number of slices along the Y-axis of the 2D atlas grid.
*   `sortOrder` (SFString): Determines tiling order. Values are `"ROW"` (default) or `"CHANNEL"`.
*   `channels` (SFString): Specifies which color channels store data. Default is `"R"`. Can include `"G"`, `"B"`, `"A"` to store larger datasets or gradient vectors.
*   `containerField` (SFNode): Targets the input field of the parent node (e.g., `voxels` for `X3DVolumeDataNode` or `surfaceNormals` for `X3DVolumeStyleNode`) [S1].

**Numbered Steps:**

1.  **Calculate Grid Dimensions:** Determine `slicesOverX` and `slicesOverY` such that `slicesOverX * slicesOverY >= numberOfSlices`. The slices are arranged in a matrix configuration within a single 2D image [S1].
2.  **Pack Slices into Atlas:**
    *   If `sortOrder` is `"ROW"`: Tile the 2D slices that represent the Z-axis of the volume data contiguously in the row of the matrix. The next slice in the Z-axis is the contiguous slice in the row [S1].
    *   If `sortOrder` is `"CHANNEL"`: Store the next Z-axis slice in the contiguous color channel. This allows multiple slices to be stored in the R, G, B, and A channels of the same 2D pixel coordinates [S1].
    *   If `channels` specifies multiple channels (e.g., `"RGBA"`), the data is distributed across these channels. For surface normals, the gradient vector (3-component) is encoded into RGB channels: R for X, G for Y, B for Z [S1].
3.  **Generate Texture:** Create the 2D texture from the packed image data. In web contexts, this can be done via the 2D Canvas API [S1].
4.  **Shader Sampling (Emulation):** In the fragment shader, resample the 2D texture to simulate 3D texture sampling. Perform trilinear interpolation at the pixel level to approximate 3D volume rendering [S1].
5.  **Coordinate Mapping:** Map the 3D volume coordinate $(u, v, w)$ to the 2D atlas coordinate $(u_{atlas}, v_{atlas})$ based on the `sortOrder` and `channels` configuration.

**Formulas:**

For `sortOrder = "ROW"`:
Let $N$ be `numberOfSlices`, $W$ be `slicesOverX`, $H$ be `slicesOverY`.
Let $z$ be the slice index ($0 \le z < N$).
The row index $r$ and column index $c$ in the atlas grid are:
$$ r = \lfloor z / W \rfloor $$
$$ c = z \mod W $$
The 2D texture coordinates $(u_{atlas}, v_{atlas})$ for a voxel at $(u, v, w)$ where $w$ is the normalized depth ($0 \le w < 1$) and slice index $z = \lfloor w \cdot N \rfloor$:
$$ u_{atlas} = \frac{c + u}{W \cdot \text{width}} $$
$$ v_{atlas} = \frac{r + v}{H \cdot \text{height}} $$
*(Note: Exact pixel-to-normalized coordinate mapping depends on the specific texture implementation, but the logical grid mapping is as above.)*

For `sortOrder = "CHANNEL"`:
The slice index $z$ determines the channel. If `channels` = "RGBA", then:
$$ \text{channel\_index} = z \mod 4 $$
The data for slice $z$ is stored in the channel corresponding to `channel\_index` (0=R, 1=G, 2=B, 3=A) at the same 2D coordinate $(u, v)$ for all slices packed in that channel.

**Parameters:**
*   `sortOrder`: `"ROW"` (default) or `"CHANNEL"` [S1].
*   `channels`: `"R"` (default), or combinations of `"R"`, `"G"`, `"B"`, `"A"` [S1].
*   `WebGL API version`: Designed for WebGL 1.0 compatibility, but valid for WebGL 2.0 [S1].

**Edge Cases and Pitfalls:**
*   **GPU Texture Size Limit:** The amount of volume data is restricted by the maximum 2D texture size of the GPU. Large datasets may require splitting or using multiple channels [S1].
*   **Precision Loss:** When using video encoding for 4D rendering, there is potential loss of accuracy due to compression and refresh time of texture data [S1].
*   **Interpolation Artifacts:** Trilinear interpolation on a 2D atlas requires careful handling of slice boundaries to avoid artifacts between adjacent slices in the matrix [S1].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def pack_atlas(slices, slices_over_x, slices_over_y, sort_order="ROW", channels="R"):
    """
    Packs 2D slices into a single 2D texture atlas.
    
    Args:
        slices: List of 2D numpy arrays (H, W) representing volume slices.
        slices_over_x: Number of slices along X axis of the atlas grid.
        slices_over_y: Number of slices along Y axis of the atlas grid.
        sort_order: "ROW" or "CHANNEL".
        channels: String of characters 'R', 'G', 'B', 'A'.
        
    Returns:
        atlas: 2D numpy array (H_atlas, W_atlas, 4) representing the packed texture.
    """
    num_slices = len(slices)
    h, w = slices[0].shape
    h_atlas = h * slices_over_y
    w_atlas = w * slices_over_x
    
    # Initialize atlas with zeros. Channels are R, G, B, A.
    atlas = np.zeros((h_atlas, w_atlas, 4), dtype=np.float32)
    
    channel_map = {'R': 0, 'G': 1, 'B': 2, 'A': 3}
    active_channels = [channel_map[c] for c in channels]
    
    for z, slice_data in enumerate(slices):
        if sort_order == "ROW":
            # Calculate grid position
            row = z // slices_over_x
            col = z % slices_over_x
            
            y_start = row * h
            x_start = col * w
            
            # Place slice in the first active channel (or R if only one)
            # Note: The source implies ROW order tiles slices in the matrix.
            # If multiple channels are used with ROW, it's ambiguous.
            # Assuming ROW primarily uses the spatial grid, and CHANNEL uses color channels.
            # If channels > 1 and ROW, we might need to define behavior.
            # Based on source: "ROW" tiles Z-axis slices contiguously in the row.
            # "CHANNEL" stores next Z-axis slice in contiguous color channel.
            # So if ROW is used, we likely only use one channel per slice location?
            # Or we stack channels? The source says "When multiple color channels are defined, 
            # the order... must be set in sortOrder".
            # Let's assume if ROW is used, we place the slice in the grid.
            # If multiple channels are specified with ROW, it's not explicitly detailed 
            # how they interleave. We'll assume ROW uses the first available channel 
            # or requires CHANNEL mode for multi-channel.
            # However, Listing 4 shows sortOrder and channels together.
            # Let's assume ROW places the slice in the grid, and if multiple channels,
            # it might repeat or we default to R.
            # To be safe, we'll place in R.
            atlas[y_start:y_start+h, x_start:x_start+w, 0] = slice_data
            
        elif sort_order == "CHANNEL":
            # Store slice in the next available color channel
            # If z >= len(channels), we might wrap or error.
            # Source: "next slice of the atlas in the Z axis direction is stored in 
            # the contiguous color channel."
            if z < len(active_channels):
                ch_idx = active_channels[z]
                # Place slice in all 2D coordinates for this channel
                # This implies the atlas is effectively a stack of 2D textures 
                # overlaid in color channels.
                atlas[:, :, ch_idx] = slice_data
            else:
                # Not enough channels defined for all slices
                pass
                
    return atlas

def sample_atlas(atlas, u, v, w, slices_over_x, slices_over_y, sort_order="ROW", channels="R"):
    """
    Samples the 2D atlas to simulate 3D texture sampling.
    
    Args:
        atlas: The packed 2D texture.
        u, v: Normalized 2D coordinates within a slice (0-1).
        w: Normalized depth coordinate (0-1).
        slices_over_x, slices_over_y: Grid dimensions.
        sort_order, channels: Same as packing.
        
    Returns:
        value: Sampled value.
    """
    num_slices = atlas.shape[0] // (atlas.shape[0] // slices_over_y) # Infer from atlas size?
    # Better to pass num_slices or infer from slices_over_x/y and atlas size
    # Let's assume we know num_slices or it's derived from the atlas dimensions
    # For this pseudocode, we assume num_slices is known or passed.
    # If not, we can't fully implement without knowing the original volume size.
    # [G] Assuming num_slices is passed or derived from the context of the volume.
    
    # Calculate slice index from w
    z = int(w * num_slices)
    if z >= num_slices:
        z = num_slices - 1
        
    if sort_order == "ROW":
        row = z // slices_over_x
        col = z % slices_over_x
        
        h, w_dim = atlas.shape[0] // slices_over_y, atlas.shape[1] // slices_over_x
        y_start = row * h
        x_start = col * w_dim
        
        # Sample from the atlas
        # Map u, v to local coordinates within the slice
        local_u = u * w_dim
        local_v = v * h
        
        atlas_u = x_start + local_u
        atlas_v = y_start + local_v
        
        # Normalize to 0-1 for texture sampling
        norm_u = atlas_u / atlas.shape[1]
        norm_v = atlas_v / atlas.shape[0]
        
        # Sample R channel (default)
        value = atlas[int(norm_v * atlas.shape[0]), int(norm_u * atlas.shape[1]), 0]
        
    elif sort_order == "CHANNEL":
        # The slice is stored in a specific channel
        if z < len(channels):
            ch_idx = {'R': 0, 'G': 1, 'B': 2, 'A': 3}[channels[z]]
            # Sample directly from the channel at u, v
            # Since all slices in a channel share the same 2D coordinates
            norm_u = u
            norm_v = v
            value = atlas[int(norm_v * atlas.shape[0]), int(norm_u * atlas.shape[1]), ch_idx]
        else:
            value = 0
            
    return value
```

## Subtopics

### 4D Volume Rendering
A technique for visualizing volume data evolving over time by creating an atlas for each time step, encoding them into a video loop, and updating the texture in real-time via the canvas API. It uses a hidden HTML5 `<video>` element to drive the texture updates [S1].

### Surface Normal Packing
A method to store surface normals (gradient vectors) in the RGB channels of the atlas. R for X, G for Y, B for Z. This allows normal-based lighting calculations in volume rendering [S1].

### Dynamic Transfer Function
Using the 2D canvas API to modify the transfer function texture directly, enabling native HTML5 transfer function editors. The TF is stored as a 2D texture [S1].

## Operating characteristics
*   **Time Complexity:** Packing is $O(N)$ where $N$ is the number of voxels, as each voxel is copied once. Sampling is $O(1)$ per pixel, but requires additional arithmetic to map 3D to 2D coordinates.
*   **Memory Complexity:** The memory usage is proportional to the size of the 2D texture. For `sortOrder="ROW"`, the texture size is roughly $W_{atlas} \times H_{atlas} \times 4$ bytes. For `sortOrder="CHANNEL"`, the texture size is $W \times H \times 4$ bytes, but only if slices fit in the channels. If more slices than channels, the method fails or requires multiple atlases.
*   **Measured Performance:** Not given in the sources.
*   **Scaling:** Limited by GPU 2D texture size limits. Larger datasets require splitting or using multiple channels [S1].
*   **Strengths:** Enables 3D volume rendering on WebGL 1.0 hardware. Allows dynamic updates via Canvas API.
*   **Limitations:** Precision loss with video encoding. Limited by 2D texture size. Interpolation artifacts between slices.
*   **When to use:** When targeting WebGL 1.0 or when dynamic texture updates via Canvas are needed.
*   **When not:** When WebGL 2.0 is available (use native 3D textures). When high precision is required and video encoding is used.

## Background
The method was proposed by Congote et al. (2011) to overcome the limitation of WebGL 1.0 not supporting 3D textures (`Texturing3D` component) [S1]. It was integrated into the X3D volume rendering component for X3DOM, a web-based GPU accelerated volume rendering library. The technique allows for interactive visualization of volumetric data (MRI/CT) in web browsers [S1].

## Sources
[S1] Community-driven extensions to the X3D volume rendering component, https://doi.org/10.1145/3055624.3075945, fetched 2026-11-02
