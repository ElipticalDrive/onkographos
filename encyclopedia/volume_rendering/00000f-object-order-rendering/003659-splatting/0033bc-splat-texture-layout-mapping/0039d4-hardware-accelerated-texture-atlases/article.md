# Hardware Accelerated Texture Atlases

Also called: Indirect Texture Sampling for Splatting, Texture Atlas Splatting

## Overview
This method organizes per-primitive texture data into large 2D atlas textures to leverage GPU hardware filtering and caching, thereby avoiding multiple texture bindings and software-based sampling overhead [S1]. It sets itself apart from software-based interpolation by utilizing dedicated texture units for bilinear filtering, which is significantly faster than manual interpolation from global memory arrays [S1].

## How it works
**Inputs and Outputs:**
*   **Inputs:** $P$ splats (primitives), $C$ total attribute channels, fixed primitive texture resolution $T$ (typically a power of 2), and GPU maximum atlas dimensions.
*   **Outputs:** One or more 2D atlas textures containing packed primitive textures, and an indirection buffer mapping primitive IDs to atlas coordinates.

**Data Structures:**
1.  **Atlas Texture(s):** Large 2D textures. If $C > 4$ (CUDA limit), split into multiple atlases (e.g., one for albedo/roughness, another for normals/metallic) [S1].
2.  **Indirection Buffer:** A buffer storing the top-left corner coordinates $(chart\_idx\_x, chart\_idx\_y)$ for each primitive's texture within the atlas [S1].

**Steps:**

1.  **Determine Atlas Geometry:**
    Calculate the maximum number of $T \times T$ textures that fit along the width and height of the atlas based on GPU capabilities. Let $N_{chartsx}$ and $N_{chartsy}$ be the total number of charts along the width and height, respectively [S1].

2.  **Pack Textures:**
    Organize the $P \times C \times T \times T$ texel data into the atlas. Each primitive's texture occupies a $T \times T$ block. The top-left corner of primitive $k$'s texture is stored in the indirection buffer at index $k$ [S1].

3.  **Sampling (Per Primitive $k$ at pixel $(x,y)$):**
    a. Map local coordinates $(u, v)$ to normalized texture coordinates $(s, t) \in [0, 1] \times [0, 1]$ [S1].
    b. Retrieve $(chart\_idx\_x, chart\_idx\_y)$ from the indirection buffer for primitive $k$ [S1].
    c. Compute atlas coordinates $(s_{atlas}, t_{atlas})$ using the formula:
       $$
       s_{atlas} = \frac{chart\_idx\_x \cdot T + s \cdot T}{N_{chartsx} \cdot T}
       $$
       $$
       t_{atlas} = \frac{chart\_idx\_y \cdot T + t \cdot T}{N_{chartsy} \cdot T}
       $$
       [S1]
    d. Sample the atlas using hardware-accelerated bilinear filtering:
       $$
       a_k(x, y) = \text{BilinearFilter}(atlas, s_{atlas}, t_{atlas})
       $$
       [S1]

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def build_atlas(primitives, T, max_atlas_width, max_atlas_height):
    """
    primitives: list of P objects, each having a T x T texture data array.
    T: resolution of each primitive's texture (power of 2).
    max_atlas_width, max_atlas_height: GPU limits in texels.
    Returns: atlas_texture, indirection_buffer
    """
    # Calculate how many T x T charts fit in the atlas
    charts_per_row = max_atlas_width // T
    charts_per_col = max_atlas_height // T
    total_charts_per_atlas = charts_per_row * charts_per_col
    
    # Determine number of atlases needed if P > total_charts_per_atlas
    # For simplicity, assume P fits in one atlas or handle multiple atlases
    # by splitting primitives. Here we assume single atlas for logic clarity.
    atlas_width = charts_per_row * T
    atlas_height = charts_per_col * T
    
    atlas = create_2d_texture(atlas_width, atlas_height)
    indirection_buffer = []
    
    for k, prim in enumerate(primitives):
        if k >= total_charts_per_atlas:
            raise ValueError("Primitive count exceeds atlas capacity")
        
        chart_idx_x = k % charts_per_row
        chart_idx_y = k // charts_per_row
        
        # Store top-left chart coordinates
        indirection_buffer.append((chart_idx_x, chart_idx_y))
        
        # Copy primitive texture to atlas block
        for u in range(T):
            for v in range(T):
                # Map local (u,v) to atlas coordinates
                atlas_x = chart_idx_x * T + u
                atlas_y = chart_idx_y * T + v
                atlas.set_texel(atlas_x, atlas_y, prim.texture[u, v])
                
    return atlas, indirection_buffer

def sample_atlas(atlas, indirection_buffer, k, s, t, T, Nchartsx, Nchartsy):
    """
    Sample the texture for primitive k at normalized coordinates (s, t).
    """
    chart_idx_x, chart_idx_y = indirection_buffer[k]
    
    # Compute atlas coordinates
    s_atlas = (chart_idx_x * T + s * T) / (Nchartsx * T)
    t_atlas = (chart_idx_y * T + t * T) / (Nchartsy * T)
    
    # Hardware bilinear filter
    return atlas.bilinear_filter(s_atlas, t_atlas)
```

**Parameters:**
*   $T$: Fixed resolution of each primitive's small texture, typically in powers of 2 [S1].
*   $N_{chartsx}, N_{chartsy}$: Total number of charts along the width and height of the atlas [S1].

**Edge Cases and Pitfalls:**
*   CUDA texture objects support up to 4 channels. If total attribute channels $C$ exceed 4, attributes must be split across multiple atlases to avoid data loss or incorrect sampling (e.g., one atlas for albedo/roughness, another for normals/metallic) [S1].

## Subtopics

### Hardware Accelerated Texture Sampling
This variant refers specifically to the sampling phase where hardware bilinear filtering is used via CUDA’s `tex2D()` intrinsic, as opposed to software interpolation. It relies on the indirection buffer to map primitive IDs to atlas chart coordinates and computes normalized atlas coordinates $(s_{atlas}, t_{atlas})$ for sampling. It is chosen when performance is critical and the GPU supports efficient texture filtering.

## Operating characteristics
*   **Time Complexity:** Sampling is $O(1)$ per primitive per pixel due to hardware filtering, significantly faster than software-based interpolation [S1].
*   **Memory Complexity:** Requires storing $P \times C \times T \times T$ texels in atlas textures plus the indirection buffer [S1].
*   **Performance:** Leverages dedicated texture caching and filtering hardware, speeding up attribute fetching compared to manual interpolation from global memory arrays [S1]. Avoids multiple texture bindings, reducing API overhead [S1].
*   **Strengths:** High performance on non-specular scenes; can reach performance levels of vanilla 3DGS on NeRF Synthetic dataset [S1].
*   **Limitations:** Limited by GPU maximum atlas dimensions and channel support (4 channels per texture object) [S1].

## Background
This technique is detailed in the supplementary material of "TextureSplat: Per-Primitive Texture Mapping for Reflective Gaussian Splatting" [S1]. It builds on the concept of Ewa splatting [S1].

## Sources
[S1] TextureSplat: Per-Primitive Texture Mapping for Reflective Gaussian Splatting, https://hal.science/hal-05484246v1, fetched 2026-10-26
