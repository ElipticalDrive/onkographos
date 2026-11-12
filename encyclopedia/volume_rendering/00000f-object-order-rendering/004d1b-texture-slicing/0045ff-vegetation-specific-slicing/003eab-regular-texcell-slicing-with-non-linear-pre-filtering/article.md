# Regular Texcell Slicing with Non-Linear Pre-filtering

Also called: Regular texcells, Slices parallel to ground

## Overview
Regular Texcell Slicing is a vegetation-specific volume rendering technique that represents forest canopies as a stack of ground-parallel planar slices (texcells) stored in a 2D texture pyramid [S1]. It solves the problem of efficient Level of Detail (LOD) management and filtering for large-scale forest scenes by avoiding CPU-side polygon generation and the artifacts associated with 3D texture MIP-mapping at grazing angles [S1]. Unlike sibling methods that may use explicit geometry or 3D textures, this method relies on pre-computed 2D alpha-premultiplied textures blended offline to create coarser LODs, ensuring correct transparency accumulation and efficient hardware rendering [S1].

## How it works
**Inputs and Outputs:**
*   **Input:** A 3D polygonal representation of a forest patch (geometry).
*   **Output:** A set of MIP-mapped 2D textures organized in a LOD pyramid, ready for GPU rendering.

**Data Structures:**
The data is stored as a pyramid of texture sets $S_i$, where $i$ ranges from $0$ (coarsest) to $N$ (finest).
*   **Level $S_N$ (Finest):** Contains $n = 2^N$ slices. Each slice is a 2D texture of size $L \times L$.
*   **Level $S_i$ (Coarser):** Contains $2^i$ slices. Each slice is a 2D texture of size $L \cdot 2^{N-i} \times L \cdot 2^{N-i}$.
*   **Texture Format:** Alpha-premultiplied RGBA. Color components $C$ store $C \times A$ [S1].
*   **MIP-mapping:** Each 2D texture is MIP-mapped to handle filtering within the slice plane [S1].

**Algorithm Steps:**

1.  **Generate Finest Level ($S_N$):**
    *   Render the original 3D forest geometry using an orthographic camera looking down (along the vertical vector $\vec{z}$).
    *   For each of the $n = 2^N$ slices, set the near and far clipping planes at distance $\delta/2$ above and below the slice's center location, where $\delta$ is the vertical spacing between slices [S1].
    *   Store the resulting rendered images as the textures in $S_N$ [S1].

2.  **Generate Coarser Levels ($S_i$ for $i < N$):**
    *   Iterate from $i = N-1$ down to $0$.
    *   For each slice index $j$ in level $S_i$:
        *   Retrieve the two corresponding slices from the finer level $S_{i+1}$: $Tex_{front} = S_{i+1}[2j]$ and $Tex_{back} = S_{i+1}[2j+1]$.
        *   Blend these two textures using the standard transparency equation for alpha-premultiplied colors:
            $$ C_{result} = \bar{C}_{front} + (1 - A_{front}) \cdot \bar{C}_{back} $$
            where $\bar{C}$ denotes the alpha-premultiplied color [S1].
        *   Store the result as $S_i[j]$.
    *   Apply MIP-mapping to each newly generated 2D texture $S_i[j]$ [S1].

**Rendering Parameters and Adjustments:**

*   **Grazing Angle Opacity Adjustment:**
    At grazing view angles, the distance between slices along the view ray ($\delta_v$) increases, but the same number of slices are traversed. This leads to incorrect total opacity if not scaled. The opacity of a slice with thickness $\delta$ must be adjusted for a ray length $dl$ along the slice normal:
    $$ A(dl) \approx \frac{dl}{\delta} A(\delta) $$
    Values greater than 1 are clamped to 1 [S1].
    
    The distance $\delta_v$ between two slices along a ray is calculated as:
    $$ \delta_v = sz \frac{|\vec{N} \cdot \vec{z}|}{|\vec{N} \cdot \vec{V}|} $$
    where:
    *   $sz$ is the vertical scaling factor (if mapped forest height differs from sample height).
    *   $\vec{N}$ is the terrain normal.
    *   $\vec{z}$ is the vertical vector $(0,1,0)$.
    *   $\vec{V}$ is the view direction [S1].

*   **Vertex Opacity Factor:**
    To implement the opacity adjustment efficiently, a vertex opacity multiply factor $\pi_A$ is computed and interpolated across the face:
    $$ \pi_A = sz \frac{\vec{N} \cdot \vec{z}}{|\vec{N} \cdot \vec{V}|} $$
    This factor modulates the texture during rasterization [S1].

*   **Aperiodic Tiling:**
    To avoid texture repetitiveness and handle precomputed shading/shadows without rotation, at least 8 different edge-matching patterns are used [S1].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def generate_texcell_pyramid(geometry, N, L, delta):
    """
    Generates the LOD pyramid for regular texcells.
    
    Args:
        geometry: 3D polygonal forest data.
        N: Log2 of the number of finest slices (n = 2^N).
        L: Base texture width/height.
        delta: Vertical distance between slices in the finest level.
        
    Returns:
        pyramid: List of lists, where pyramid[i] contains 2^i textures.
    """
    n = 2**N
    pyramid = [None] * (N + 1)
    
    # Step 1: Generate finest level SN
    # Render n slices using orthographic camera looking down
    # Clipping planes at slice_center +/- delta/2
    finest_slices = []
    for j in range(n):
        slice_center_y = j * delta
        near_clip = slice_center_y - delta / 2
        far_clip = slice_center_y + delta / 2
        
        # Render geometry with clipping planes
        texture = render_orthographic_slice(geometry, slice_center_y, near_clip, far_clip, L)
        finest_slices.append(texture)
    
    pyramid[N] = finest_slices
    
    # Step 2: Generate coarser levels Si
    for i in range(N - 1, -1, -1):
        current_level_slices = []
        num_slices_in_level = 2**i
        num_slices_in_finer = 2**(i + 1)
        
        # Texture size for this level
        tex_size = L * (2**(N - i))
        
        for j in range(num_slices_in_level):
            # Retrieve two slices from the finer level
            tex_front = pyramid[i + 1][2 * j]
            tex_back = pyramid[i + 1][2 * j + 1]
            
            # Blend using transparency equation for alpha-premultiplied colors
            # C_result = C_bar_front + (1 - A_front) * C_bar_back
            blended_tex = blend_alpha_premultiplied(tex_front, tex_back)
            
            # MIP-map the resulting 2D texture
            mipmapped_tex = generate_mipmap(blended_tex, tex_size)
            current_level_slices.append(mipmapped_tex)
            
        pyramid[i] = current_level_slices
        
    return pyramid

def render_orthographic_slice(geometry, center_y, near_clip, far_clip, resolution):
    """
    Renders a single slice of geometry using an orthographic camera.
    Returns an alpha-premultiplied RGBA texture.
    """
    # Implementation depends on specific renderer API, but conceptually:
    # 1. Set camera projection to orthographic, looking down -Z or +Y.
    # 2. Set view frustum near plane to near_clip, far plane to far_clip.
    # 3. Render geometry.
    # 4. Capture frame buffer as RGBA texture.
    # 5. Ensure color components are premultiplied by alpha.
    pass

def blend_alpha_premultiplied(tex1, tex2):
    """
    Blends two alpha-premultiplied textures.
    tex1 is 'front', tex2 is 'back'.
    """
    # For each pixel (u,v):
    # C1 = tex1.color, A1 = tex1.alpha
    # C2 = tex2.color, A2 = tex2.alpha
    # C_out = C1 + (1 - A1) * C2
    # A_out = A1 + (1 - A1) * A2
    # Return texture with C_out, A_out
    pass

def compute_opacity_factor(terrain_normal, view_direction, vertical_scaling):
    """
    Computes the vertex opacity multiply factor pi_A.
    """
    N = normalize(terrain_normal)
    V = normalize(view_direction)
    z = vector(0, 1, 0) # Vertical vector
    sz = vertical_scaling
    
    numerator = dot(N, z)
    denominator = abs(dot(N, V))
    
    # Avoid division by zero if view is exactly parallel to terrain normal (unlikely in practice for slicing)
    if denominator < epsilon:
        return 1.0
        
    pi_A = sz * (numerator / denominator)
    return pi_A
```

## Subtopics

### Silhouette Texcell Slicing
A variant used for landscape silhouettes where slices face the viewer (grazing angles). It uses a tilted base triangle offset to generate slices on the GPU, avoiding CPU polygon generation [S1]. The base triangle is adjusted to be just below the ground ($P'_i = P_i + (k_i - k_{max})\vec{z}$) to ensure the whole volume is sampled, and slices are generated by offsetting this base [S1]. It uses 3D textures where the MIP-map level is adapted to the slicing rate to mitigate blurring, though 3D texture MIP-mapping has known implementation bugs [S1].

## Operating characteristics
*   **Time Complexity:** Pre-processing is $O(n \cdot L^2)$ for generating the finest level and $O(n \cdot L^2)$ for blending coarser levels (since each level halves the number of slices but doubles the texture size, the total work per level is constant, and there are $N$ levels). Rendering is $O(1)$ per texcell per LOD level, with LOD selection based on distance.
*   **Memory Complexity:** $O(n \cdot L^2)$ for the finest level. The total memory for the pyramid is roughly $2 \cdot n \cdot L^2$ (geometric series sum), which is manageable.
*   **Performance:** High performance for large-scale forests. Avoids CPU-GPU transfer bottlenecks by generating slices via clipping planes in a single pass (or offline). Hardware MIP-mapping and anisotropic filtering (for 2D textures) are efficient [S1].
*   **Strengths:** Efficient LOD management, correct filtering in the slice plane, avoids 3D texture MIP-map bugs, handles precomputed shading well with aperiodic tiling [S1].
*   **Limitations:** Poor quality at grazing angles (parallax error, visible gaps between slices) unless switched to Silhouette Texcell Slicing. Requires offline rendering or significant GPU time to generate the initial slices. Not suitable for dynamic geometry changes [S1].
*   **When to use:** Static or semi-static forest scenes viewed from a distance or normal incidence angles.
*   **When not to use:** Close-up views of silhouettes, dynamic vegetation, or scenes requiring high-quality filtering in the slicing direction at grazing angles (use Silhouette Texcell Slicing instead) [S1].

## Background
The method was introduced in the paper "Rendering Forest Scenes in Real-Time" [S1]. It builds on the concept of texcells (texture cells) introduced by [KK89] and uses real-time volumetric textures [MN98, LPFH01]. The approach addresses issues with isotropic filtering in 3D textures at grazing angles and the bugs present in 3D texture MIP-mapping implementations on contemporary hardware (as of 2004) [S1].

## Sources
[S1] Rendering Forest Scenes in Real-Time, https://doi.org/10.2312/egwr/egsr04/093-102, fetched 2026-10-26
