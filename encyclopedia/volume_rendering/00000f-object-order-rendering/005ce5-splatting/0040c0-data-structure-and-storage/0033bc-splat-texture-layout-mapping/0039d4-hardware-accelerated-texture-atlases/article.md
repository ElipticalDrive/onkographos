# Hardware Accelerated Texture Atlases

Also called: Indirect Texture Sampling for Splatting, Texture Atlas Splatting

## Overview
Hardware Accelerated Texture Atlases organize per-primitive texture data into large 2D atlas textures to leverage GPU hardware filtering and caching, thereby avoiding multiple texture bindings and software-based sampling overhead [S1]. This method builds on [Splat Texture Layout & Mapping](7089-splat-texture-layout-mapping.md) by consolidating individual primitive textures into a unified structure that enables efficient, filtered attribute fetching during rendering [S1].

## How it works
**Inputs and Outputs:**
*   **Inputs:** $P$ splats, each associated with a small fixed-resolution texture of size $T \times T$ texels; $C$ total attribute channels across all splats; GPU maximum texture dimensions.
*   **Outputs:** One or more large 2D atlas textures; an indirection buffer mapping primitive IDs to atlas coordinates.

**Data Structures:**
1.  **Atlas Texture(s):** Large 2D textures containing packed $T \times T$ charts. The layout is determined by GPU capabilities. If the total channels $C$ exceed the maximum supported by a single CUDA texture object (typically 4), multiple atlases are created [S1]. For example, one atlas may store Albedo (RGB) and Roughness (A), while another stores Tangent Normals (RGB) and Metallic (A) [S1].
2.  **Indirection Buffer:** A buffer storing the top-left corner coordinates $(chart\_idx\_x, chart\_idx\_y)$ for each primitive's texture chart within the atlas [S1]. This maps primitive IDs to their specific location in the atlas grid.

**Parameters:**
*   $T$: Fixed resolution of each primitive's small texture, typically a power of 2 [S1].
*   $N_{chartsx}, N_{chartsy}$: Total number of charts along the width and height of the atlas, respectively [S1].

**Algorithm Steps:**

1.  **Determine Atlas Dimensions:** Calculate the maximum atlas dimensions supported by the GPU. Determine how many $T \times T$ textures fit along each dimension ($N_{chartsx}, N_{chartsy}$) [S1].
2.  **Calculate Atlas Count:** Compute the total number of textures per atlas based on channel constraints (e.g., max 4 channels per CUDA texture object). Determine the total number of atlases needed to store all $P \times C$ textures [S1].
3.  **Construct Atlases:** Pack the $P \times C \times T \times T$ texel data into the atlas textures. Assign each primitive's texture to a specific chart position within the atlas [S1].
4.  **Build Indirection Buffer:** For each primitive $k$, store the top-left chart coordinates $(chart\_idx\_x_k, chart\_idx\_y_k)$ in the indirection buffer [S1].

**Rendering (Hardware Accelerated Texture Sampling):**
For a given primitive $k$ and local coordinates $(u, v)$:

1.  **Normalize Coordinates:** Map local coordinates to normalized texture coordinates $(s, t) \in [0, 1] \times [0, 1]$ [S1].
2.  **Retrieve Chart Coordinates:** Look up $(chart\_idx\_x, chart\_idx\_y)$ for primitive $k$ from the indirection buffer [S1].
3.  **Compute Atlas Coordinates:** Calculate the normalized coordinates within the atlas texture using the following formulas [S1]:
    $$
    s_{atlas} = \frac{chart\_idx\_x \cdot T + s \cdot T}{N_{chartsx} \cdot T}
    $$
    $$
    t_{atlas} = \frac{chart\_idx\_y \cdot T + t \cdot T}{N_{chartsy} \cdot T}
    $$
4.  **Sample:** Perform hardware-accelerated bilinear filtering using the CUDA `tex2D()` intrinsic:
    $$
    a_k(x, y) = \text{BilinearFilter}(atlas, s_{atlas}, t_{atlas})
    $$
    [S1]

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def construct_atlas(primitives, T, max_atlas_width, max_atlas_height, max_channels_per_atlas):
    """
    Constructs texture atlases and indirection buffer.
    primitives: List of primitive objects, each having a texture data array of size T x T x C_k
    T: Texture resolution per primitive (power of 2)
    max_atlas_width, max_atlas_height: Max dimensions supported by GPU
    max_channels_per_atlas: Max channels per CUDA texture object (typically 4)
    """
    # Calculate chart capacity per atlas
    charts_per_atlas_x = max_atlas_width // T
    charts_per_atlas_y = max_atlas_height // T
    charts_per_atlas = charts_per_atlas_x * charts_per_atlas_y
    
    # Determine total atlases needed based on total channels
    total_channels = sum(p.channels for p in primitives)
    atlases_per_primitive = ceil(total_channels / primitives[0].channels) # Simplified assumption for uniform C
    # More accurately: Group primitives by channel count or split attributes
    # For this implementation, assume we split attributes across atlases if total C > max_channels_per_atlas
    # The source implies splitting by attribute type (e.g., RGB+A vs RGB+A)
    
    # For simplicity, assume we pack as many charts as possible into one atlas
    # If total channels exceed max_channels_per_atlas, we create multiple atlases
    # Let's assume a single atlas holds up to max_channels_per_atlas channels worth of data
    # Number of atlases = ceil(Total_Texture_Units / Charts_Per_Atlas)
    # Where Total_Texture_Units = P * ceil(C / max_channels_per_atlas)
    
    num_textures_per_primitive = ceil(primitives[0].channels / max_channels_per_atlas)
    total_textures = len(primitives) * num_textures_per_primitive
    num_atlases = ceil(total_textures / charts_per_atlas)
    
    atlases = []
    indirection_buffer = []
    
    current_atlas_idx = 0
    current_chart_idx = 0
    
    for p_idx, primitive in enumerate(primitives):
        # Determine which atlas and chart this primitive's texture belongs to
        atlas_idx = current_atlas_idx
        chart_idx = current_chart_idx
        
        # Calculate chart coordinates (x, y) within the atlas
        chart_x = chart_idx % charts_per_atlas_x
        chart_y = chart_idx // charts_per_atlas_x
        
        # Store indirection info
        indirection_buffer.append({
            'primitive_id': p_idx,
            'atlas_idx': atlas_idx,
            'chart_idx_x': chart_x,
            'chart_idx_y': chart_y
        })
        
        # Place texture data into atlas (implementation detail omitted, assumes atlas creation)
        # ... pack primitive.texture_data into atlases[atlas_idx] at position (chart_x * T, chart_y * T) ...
        
        current_chart_idx += 1
        if current_chart_idx >= charts_per_atlas:
            current_chart_idx = 0
            current_atlas_idx += 1
            
    return atlases, indirection_buffer

def sample_texture(primitive_idx, u, v, atlases, indirection_buffer, T, N_charts_x, N_charts_y):
    """
    Samples texture for a primitive using hardware acceleration.
    """
    # 1. Normalize local coordinates
    s = u
    t = v
    
    # 2. Retrieve chart coordinates from indirection buffer
    info = indirection_buffer[primitive_idx]
    chart_x = info['chart_idx_x']
    chart_y = info['chart_idx_y']
    atlas_idx = info['atlas_idx']
    
    # 3. Compute atlas coordinates
    # Formula: (chart_idx * T + s * T) / (N_charts * T)
    # Simplifies to: (chart_idx + s) / N_charts
    s_atlas = (chart_x + s) / N_charts_x
    t_atlas = (chart_y + t) / N_charts_y
    
    # 4. Sample using hardware bilinear filtering
    # In CUDA: tex2D(atlas_texture_object, s_atlas, t_atlas)
    # Here we simulate the result
    atlas = atlases[atlas_idx]
    color = hardware_bilinear_filter(atlas, s_atlas, t_atlas)
    
    return color
```

## Subtopics

### Hardware Accelerated Texture Sampling
A rendering technique where texture sampling is performed using GPU hardware filtering (e.g., CUDA `tex2D()`) rather than software interpolation. It maps primitive local coordinates to normalized atlas coordinates and retrieves values via bilinear filtering, leveraging dedicated texture caching for significant performance gains over manual global memory access [S1].

### Multi-Texturing for Splat Atlases
A process applied after shape/location refinement to generate a single texture atlas per splat by blending color information from multiple cameras. Unlike Yang et al. [2006] which uses view-dependent dynamic texture selection, or Sibbing et al. [2013] which uses a single texture per splat with three-pass rendering, this method blends multiple images into one atlas per splat to reduce inaccuracies from misalignments or irregular lighting [S2]. It works for splats even when they partially occlude each other, producing seamless transitions across surfaces [S2].

## Operating characteristics
*   **Performance:** Significantly faster than software-based sampling approaches due to the use of dedicated texture caching and filtering hardware [S1]. Improves performance by avoiding multiple texture bindings during rendering, reducing API overhead [S1].
*   **Memory:** Requires large 2D atlas textures. The number of atlases depends on GPU texture object channel limits (typically 4) and total attribute channels [S1].
*   **Limitations:** CUDA texture objects typically support up to 4 channels; exceeding this requires splitting attributes across multiple atlases, which increases complexity [S1].
*   **Use Cases:** Suitable for non-specular scenes where high-quality material and normal texture splatting is required, capable of reaching performance levels comparable to vanilla 3DGS on NeRF Synthetic datasets [S1].

## Background
The technique is detailed in "TextureSplat: Per-Primitive Texture Mapping for Reflective Gaussian Splatting" [S1]. It builds upon the concept of Ewa splatting [Gross 2002] [S1]. The multi-texturing aspect is derived from work on textured splat-based point clouds for handheld devices [S2], which itself references Pagés et al. [2015] for the blending approach [S2].

## Sources
[S1] TextureSplat: Per-Primitive Texture Mapping for Reflective Gaussian Splatting, https://hal.science/hal-05484246v1, fetched 2026-10-26
[S2] Textured splat-based point clouds for rendering in handheld devices, https://doi.org/10.1145/2775292.2782779, fetched 2026-11-03
