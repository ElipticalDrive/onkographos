# Triplanar Texture Projection

Also called: Round Cube Mapping, Trianplanar Texture Projection, Triplanar Mapping, Triplanar Texture Mapping

## Overview
Triplanar texture projection maps 2D textures onto 3D surfaces by sampling along three orthogonal planes (XY, XZ, YZ) and blending the results based on the surface normal [S1, unofficial][S2][S3, unofficial][S4][S5, unofficial]. It solves the problem of texture stretching that occurs with simple planar mapping on complex or steep surfaces, such as terrain with large altitude differences [S1, unofficial][S2, S4][S5, unofficial]. Unlike its sibling [Planar Projection Texture Mapping](3740-planar-projection-texture-mapping.md), which uses a single axis-aligned projection, triplanar mapping blends three projections to maintain consistent texture scale and avoid distortion regardless of surface orientation [S2, S4][S5, unofficial].

## How it works
**Inputs and Outputs:**
*   **Inputs:**
    *   `tex`: A 2D texture sampler.
    *   `p`: A 3D position vector $(p_x, p_y, p_z)$ in world or local space [S2][S3, unofficial][S4][S5, unofficial].
    *   `n`: A 3D surface normal vector $(n_x, n_y, n_z)$, typically normalized [S2][S3, unofficial][S4][S5, unofficial].
    *   `k`: A scalar "hardness" parameter controlling the sharpness of the blending weights (default $k=1$) [S2, S4][S5, unofficial].
*   **Output:**
    *   `color`: A 3D color vector resulting from the blended texture samples [S1, unofficial][S2, S4].

**Data Structures:**
No complex data structures are required. The method operates on standard vector types. The texture `tex` is assumed to be a standard 2D image with UV coordinates in $[0, 1]$ or $[-1, 1]$ depending on implementation, but the projection coordinates are derived directly from `p` [S2][S3, unofficial].

**Steps:**
1.  **Compute Absolute Normal Components:** Calculate the absolute values of the normal components: $a_x = |n_x|$, $a_y = |n_y|$, $a_z = |n_z|$ [S2, S4][S5, unofficial].
2.  **Apply Hardness (Optional):** If a hardness parameter $k$ is used, raise the absolute components to the power of $k$: $w_x = a_x^k$, $w_y = a_y^k$, $w_z = a_z^k$ [S2, S4][S5, unofficial]. If $k=1$, $w_i = a_i$.
3.  **Normalize Weights:** Calculate the sum of weights $W = w_x + w_y + w_z$. To prevent division by zero, clamp $W$ to a small epsilon if necessary [G]. Normalize the weights: $W_x = w_x / W$, $W_y = w_y / W$, $W_z = w_z / W$ [S2][S3, unofficial][S4][S5, unofficial].
4.  **Sample Textures:** Perform three texture lookups using the position `p` projected onto the respective planes. The UV coordinates for each plane are derived from the two position components *not* corresponding to the projection axis, scaled and offset to fit the texture space [S2][S3, unofficial][S4][S5, unofficial].
    *   Sample for the XY plane (weighted by Z-normal): $c_z = \text{sample}(p_x, p_y)$ [S2][S3, unofficial][S4][S5, unofficial].
    *   Sample for the XZ plane (weighted by Y-normal): $c_y = \text{sample}(p_x, p_z)$ [S2][S3, unofficial][S4][S5, unofficial].
    *   Sample for the YZ plane (weighted by X-normal): $c_x = \text{sample}(p_y, p_z)$ [S2][S3, unofficial][S4][S5, unofficial].
    *   *Note on UV Scaling:* Some implementations scale the position components by a texture scale factor and offset by 0.5 to center the texture [S3, unofficial]. The core formula assumes direct mapping or consistent scaling across all three planes.
5.  **Blend Results:** Compute the final color by weighting the samples with the normalized weights:
    $$ \text{color} = W_x c_x + W_y c_y + W_z c_z $$
    [S2, S4][S5, unofficial].

**Formulas:**
The general formula for weighted triplanar mapping is [S2, S4][S5, unofficial]:

$$ \text{texture}_{\text{triplanar}}(p, n, k) = \frac{|n_x|^k \text{sample}(p_y, p_z) + |n_y|^k \text{sample}(p_x, p_z) + |n_z|^k \text{sample}(p_x, p_y)}{|n_x|^k + |n_y|^k + |n_z|^k} $$

Where:
*   $p = (p_x, p_y, p_z)$ is the position.
*   $n = (n_x, n_y, n_z)$ is the surface normal.
*   $k$ is the hardness parameter.
*   $\text{sample}(u, v)$ is the texture sampling function.

For the unweighted case ($k=1$) [S4][S5, unofficial]:

$$ \text{texture}_{\text{triplanar}}(p, n) = \frac{|n_x| \text{sample}(p_y, p_z) + |n_y| \text{sample}(p_x, p_z) + |n_z| \text{sample}(p_x, p_y)}{|n_x| + |n_y| + |n_z|} $$

**Parameters:**
*   **k (Hardness):** Controls the alignment weighting. Higher values make the texture align more strictly with the dominant normal axis, reducing blending artifacts but potentially increasing visible seams if the normal changes abruptly. Typical values range from 1 (linear blend) to 5 or higher for sharper alignment [S2][S3, unofficial][S4][S5, unofficial]. [S3, unofficial] uses $k=5.0$ in its implementation.

**Pitfalls and Edge Cases:**
*   **Blurring at Intermediate Normals:** Textures appear blurred or washed out at points where the normal points between axis-aligned planes (e.g., 45-degree slopes), because no single blending weight dominates [S1, unofficial].
*   **Memory Bandwidth:** Triplanar mapping requires three texture lookups per shading evaluation. This makes it unsuitable for sampling heightmaps in the distance function (`map` function) during ray marching, as it incurs significantly higher memory bandwidth overhead compared to biplanar or uniplanar mapping [S2, S4][S5, unofficial].
*   **Projection Alignment:** The UV coordinates must be consistently scaled and oriented across all three planes to avoid visible seams or misalignment artifacts [S7].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def triplanar_texture_projection(p: Vec3, n: Vec3, k: float = 1.0) -> Vec3:
    """
    Perform triplanar texture projection.
    
    Args:
        p: Position vector (px, py, pz)
        n: Surface normal vector (nx, ny, nz)
        k: Hardness parameter for weighting
        
    Returns:
        Blended texture color
    """
    # 1. Compute absolute normal components
    abs_n = abs(n)
    
    # 2. Apply hardness parameter
    weights = abs_n ** k
    
    # 3. Normalize weights
    weight_sum = weights.x + weights.y + weights.z
    # Guard against division by zero
    if weight_sum < 1e-6:
        weight_sum = 1e-6
        
    normalized_weights = weights / weight_sum
    
    # 4. Sample textures for each plane
    # XY plane (projected along Z, weighted by Z normal component)
    # Note: UV coordinates are (px, py)
    sample_xy = sample_texture(p.x, p.y)
    
    # XZ plane (projected along Y, weighted by Y normal component)
    # Note: UV coordinates are (px, pz)
    sample_xz = sample_texture(p.x, p.z)
    
    # YZ plane (projected along X, weighted by X normal component)
    # Note: UV coordinates are (py, pz)
    sample_yz = sample_texture(p.y, p.z)
    
    # 5. Blend results
    # Weight for XY sample is determined by Z-normal component
    # Weight for XZ sample is determined by Y-normal component
    # Weight for YZ sample is determined by X-normal component
    color = (
        normalized_weights.z * sample_xy +
        normalized_weights.y * sample_xz +
        normalized_weights.x * sample_yz
    )
    
    return color
```

## Subtopics

### [Tri-Planar Mapping with Centroid Normals](3740-tri-planar-mapping-with-centroid-normals.md)
This variant applies triplanar texture projection to voxel-based geometry by replacing standard face-aligned normals with normals derived from the voxel's centroid. It solves the problem of hard edges and texture discontinuities that occur when distinct 2D texture slices are mapped to the six faces of a voxel cube. By using centroid-based normals, the method creates smooth material transitions and avoids the visible seams inherent in standard AABB texturing. It differs from standard triplanar mapping by dynamically computing the projection weights based on the geometric relationship between the ray intersection point and the voxel center, rather than relying on pre-computed or face-averaged normals.

### Planar Projection Texture Mapping
Maps a 2D texture to a 3D point by using two of the three coordinates of the point on the zero-isosurface to sample the texture [S5, unofficial]. The default axis is the xz-axis, used for floor plane alignment [S5, unofficial]. Formula: $\text{texture}_{\text{planar}}(p) = \text{sample}(p_x, p_z)$ [S5, unofficial].

### Projective Texturing
Textures a mesh using alignment between the mesh and an input image to estimate its UV map [S6]. Pitfall: Hidden areas copy the appearance of visible parts, even when symmetry is accounted for. Lighting is baked into the appearance, making novel lighting difficult [S6].

### Triplanar Projection for Meshes
Renders triangle meshes with automatic texture mapping for surfaces lacking inherent UV parameterization, using tri-planar projection and data-driven blending [S7]. It blends projections according to the surface normal and influences blending by a projection blending factor to control seam crispness [S7]. Low factors are preferred for structured textures (e.g., rice grains), while high factors help hide seams in continuously varying textures (e.g., watercolor strokes) [S7].

## Operating characteristics
*   **Time Complexity:** $O(1)$ per shading point, but requires three texture lookups per evaluation [S2, S4][S5, unofficial].
*   **Memory Complexity:** $O(1)$ additional memory, assuming the texture is already loaded [S2].
*   **Performance:** Suitable for shading due to the fixed maximum of three samples per camera ray [S2, S4][S5, unofficial]. However, it is not suitable for use in the distance function (`map` function) during ray marching because the repeated texture lookups per iteration cause high memory bandwidth overhead compared to simpler biplanar mapping [S2, S4][S5, unofficial].
*   **Strengths:** Prevents texture stretching on complex surfaces; works well for terrain and organic shapes without UV mapping; seamless blending when normals are smooth [S1, unofficial][S2, S4][S5, unofficial].
*   **Limitations:** Blurring at intermediate normals; high memory bandwidth cost if used in the marching loop; potential for visible seams if the projection blending factor is not tuned correctly [S1, unofficial][S2, S4][S5, unofficial][S7].

## Background
Triplanar mapping, also known as round cube mapping, was developed as a solution to the texture stretching issues inherent in simple planar projection mapping for complex surfaces like spheres and terrain [S2, S4][S5, unofficial]. It allows for procedural rendering without the need for explicit UV parameterization [S2, S4][S5, unofficial]. The technique is widely used in terrain rendering and procedural generation to apply consistent textures to steep slopes and complex geometries [S1, unofficial][S2, S4][S5, unofficial]. Variants have been adapted for voxel-based rendering to smooth out artifacts associated with axis-aligned bounding box (AABB) texturing [S1, unofficial] and for data-driven visualization on arbitrary 3D meshes [S7].

## Sources
[S1] Terrain synthesis using noise, https://trepo.tuni.fi/handle/10024/101043, fetched 2026-10-26 (unofficial)
[S2] Raymarching Distance Fields with CUDA, https://doi.org/10.3390/electronics10222730, fetched 2026-10-26
[S3] kushchevd/RayMarching, https://github.com/kushchevd/RayMarching, fetched 2026-10-26 (unofficial)
[S4] Raymarching Distance Fields with CUDA, https://www.mdpi.com/2079-9292/10/22/2730, fetched 2026-10-27
[S5] (PDF) Raymarching Distance Fields with CUDA - ResearchGate, https://www.researchgate.net/publication/356081826_Raymarching_Distance_Fields_with_CUDA, fetched 2026-10-27 (unofficial)
[S6] Neural Voxel Renderer: Learning an Accurate and Controllable Rendering Tool, https://doi.org/10.1109/cvpr42600.2020.00546, fetched 2026-10-26
[S7] Artifact-Based Rendering: Harnessing Natural and Traditional Visual Media for More Expressive and Engaging 3D Visualizations, https://doi.org/10.1109/tvcg.2019.2934260, fetched 2026-10-26
