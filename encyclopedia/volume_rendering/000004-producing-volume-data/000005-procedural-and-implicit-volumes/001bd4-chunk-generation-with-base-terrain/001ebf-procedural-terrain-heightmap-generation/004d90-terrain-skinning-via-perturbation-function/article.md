# Terrain Skinning via Perturbation Function

Also called: Base Surface + Altitude Map, Freeform Surface Terrain Representation

## Overview
Terrain Skinning via Perturbation Function represents complex terrain as a simple base surface (e.g., a plane) perturbed by a scalar altitude map, eliminating the need for polygonal triangulation [S1, S2]. Unlike voxel-based methods that require mesh extraction (e.g., Marching Cubes) or traditional heightmap rendering that suffers from LOD cracking and high depth complexity, this method uses recursive voxel subdivision and direct texture-sampling of the altitude map to modulate the base surface equation, resulting in low depth complexity and seamless rendering [S2].

## How it works
**Inputs and Outputs**
*   **Inputs:**
    *   Base surface $P$: A simple surface defined by an algebraic equation (typically a plane, ellipsoid, or cylinder) [S1, S2].
    *   Altitude map (Perturbation function $h$): A 2D rectangular table of scalar values representing height deviations, with dimensions $2^n \times 2^n$ [S2].
    *   Level of Detail (LOD) maps: Precomputed hierarchical maximum-value maps derived from the altitude map [S2].
    *   Voxel $V_0$: Defined by a near vector $\mathbf{P}_0$ and a far vector $\mathbf{P}_1$ in model space [S2].
    *   Transformation matrix $M$: The geometric transformation from model space to the base surface coordinate system [S2].
*   **Outputs:**
    *   A set of points defining the terrain surface $F$, calculated as positions on the base surface modified by the altitude map [S1, S2].

**Data Structures**
1.  **Altitude Map Table:** A 2D array of size $2^n \times 2^n$ storing scalar height values [S2].
2.  **LOD Maps:** A hierarchy of arrays. Level $n$ is the original altitude map. Level $k-1$ is derived from level $k$ by taking the maximum value of every $2 \times 2$ block, resulting in a $(2^{k-1}) \times (2^{k-1})$ array. Level 0 contains a single value (the global maximum) [S2].
3.  **Voxel:** A structure containing two 3D vectors, $\mathbf{P}_n$ (near) and $\mathbf{P}_f$ (far), representing the bounds of the voxel along the view ray or subdivision axis [S2].

**Transformation Matrix**
The transformation $G$ maps 3D model coordinates to the 2D altitude map coordinates $(u, v)$. It is composed of a geometric transformation $M$ and a projection matrix $T$ [S2]:
$$ G = T \cdot M $$
Where $T$ is defined as:
$$ T = \begin{bmatrix} 1 & 0 & 0 & 0 \\ 0 & 0 & 1 & 0 \\ 0 & 1 & 0 & 0 \\ 0 & 0 & 0 & 1 \end{bmatrix} $$
This matrix swaps the $y$ and $z$ axes, effectively projecting the 3D point onto the $u,v$ plane of the altitude map [S2].

**Algorithm Steps**
1.  **Initialize Voxel:** Start with voxel $V_0$ defined by near vector $\mathbf{P}_0$ and far vector $\mathbf{P}_1$ [S2].
2.  **Transform Coordinates:** Convert the voxel endpoints from model space $(x, y, z)$ to the height map coordinate system $(u, v, h, a)$ using matrix $G$:
    $$ \begin{bmatrix} u \\ v \\ h \\ a \end{bmatrix} = G \cdot \begin{bmatrix} x \\ y \\ z \\ 1 \end{bmatrix} $$
    Let the transformed endpoints be $(u_0, v_0, h_0, a_0)$ and $(u_1, v_1, h_1, a_1)$ [S2].
3.  **Recursive Subdivision:** Perform binary subdivision on the voxel along the Z-coordinate (or view direction). For recursion level $i$:
    *   The near vector remains unchanged from the previous level: $\mathbf{P}_{n_i} = \mathbf{P}_{n_{i-1}}$ [S2].
    *   The far vector is the midpoint of the previous near and far vectors: $\mathbf{P}_{f_i} = (\mathbf{P}_{n_{i-1}} + \mathbf{P}_{f_{i-1}}) / 2$ [S2].
4.  **Calculate Level of Detail (LOD):**
    *   Project the current voxel endpoints $\mathbf{P}_{n_i}$ and $\mathbf{P}_{f_i}$ to the altitude map coordinates to get $(u_{n_i}, v_{n_i})$ and $(u_{f_i}, v_{f_i})$ [S2].
    *   Calculate the maximum distance $L_p$ between these projected points:
        $$ L_p = \max(\text{distance}((u_{n_i}, v_{n_i}), (u_{f_i}, v_{f_i}))) $$
    *   Determine the LOD level $l$ using the inequality:
        $$ \frac{1}{2^l} < L_p < \frac{1}{2^{l+1}} $$
        This selects the appropriate resolution from the precomputed LOD maps [S2].
5.  **Sample Perturbation:**
    *   Using the LOD level $l$, sample the maximum height value $H_{max}$ from the corresponding LOD map at the coordinates of the voxel projection [S2].
    *   If the current recursion level is not the final level, compare the calculated altitude $h$ (derived from the voxel's position relative to the base plane) with $H_{max}$. If $h > H_{max}$, stop subdivision for this branch [S2].
6.  **Interpolate Height:**
    *   For the final level of detail, calculate the perturbation value $h(u, v)$ using bilinear interpolation of the altitude map table [S1, S2].
    *   Let $m_u = \lfloor u \rfloor$ and $m_v = \lfloor v \rfloor$ be the integer parts of the coordinates.
    *   Calculate intermediate values $f_0$ and $f_1$:
        $$ f_0 = (1 - (u - m_u)) \cdot \text{table}[m_u, m_v] + (u - m_u) \cdot \text{table}[m_u+1, m_v] $$
        $$ f_1 = (1 - (u - m_u)) \cdot \text{table}[m_u, m_v+1] + (u - m_u) \cdot \text{table}[m_u+1, m_v+1] $$
    *   Calculate final height $h$:
        $$ h(u, v) = f_0 + (f_1 - f_0) \cdot (v - m_v) $$
    *   If higher accuracy is required, bicubic interpolation may be used instead [S2].
7.  **Compute Surface Point:**
    *   Calculate the final position on the terrain surface $F$ using the vector equation:
        $$ F = G(\mathbf{v}) + \mathbf{n} \cdot h(G(\mathbf{v})) $$
        Where $\mathbf{v}$ is the point in 3D space, $G(\mathbf{v})$ is its projection onto the base surface, and $\mathbf{n}$ is the normal vector to the base surface [S1, S2].
    *   If $\mathbf{v}$ is outside the perturbation domain $D_P$, the term $\mathbf{n} \cdot h(G(\mathbf{v}))$ is zero, and $F$ coincides with the base surface $P$ [S1, S2].

**Pseudocode**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def terrain_skinning(p0, p1, base_normal, altitude_map, lod_maps, transform_matrix_g):
    """
    Renders terrain using recursive voxel subdivision and perturbation function.
    
    Args:
        p0: Vector3, near endpoint of the voxel in model space.
        p1: Vector3, far endpoint of the voxel in model space.
        base_normal: Vector3, normal vector of the base surface (e.g., plane).
        altitude_map: 2D array, the highest resolution height map (2^n x 2^n).
        lod_maps: List of 2D arrays, precomputed LOD maps (level n down to 0).
        transform_matrix_g: Matrix4x4, transformation G = T * M.
        
    Returns:
        List of points on the terrain surface.
    """
    points = []
    # Stack for recursion: stores (p_near, p_far, current_lod_index)
    stack = [(p0, p1, len(lod_maps) - 1)]
    
    while stack:
        p_near, p_far, lod_idx = stack.pop()
        
        # Transform endpoints to altitude map coordinates
        u0, v0, _, _ = transform_matrix_g * homogeneous(p_near)
        u1, v1, _, _ = transform_matrix_g * homogeneous(p_far)
        
        # Calculate projection distance Lp
        # Distance in UV space
        dx = u1 - u0
        dy = v1 - v0
        Lp = sqrt(dx*dx + dy*dy)
        
        # Determine LOD level l based on Lp
        # Inequality: 1/2^l < Lp < 1/2^(l+1)
        # This implies l is roughly -log2(Lp)
        # We use the provided lod_idx which corresponds to the current resolution
        # The source defines LOD selection by comparing Lp to powers of 2.
        # For implementation, we assume lod_idx is the correct level for the current voxel size.
        
        # Sample max height from LOD map
        # Map u,v to LOD map coordinates
        lod_size = 2 ** lod_idx
        u_lod = u0 * lod_size
        v_lod = v0 * lod_size
        
        # Get max height for this LOD block
        # Note: Source says "sample of maximum value from table presenting given level of detail"
        # This implies we look up the precomputed max for the region covered by the voxel
        h_max = lod_maps[lod_idx].get_max(u_lod, v_lod, u1, v1)
        
        # Calculate altitude h of the voxel center or relevant point
        # The source mentions comparing calculated height h to Hmax
        # h is the deviation from the base plane.
        # For a plane base, h can be derived from the dot product with normal
        # However, the source uses the transformation G to get h component directly in some contexts,
        # but primarily uses the altitude map for h.
        # Let's calculate the perturbation value from the altitude map for the current voxel center
        u_center = (u0 + u1) / 2
        v_center = (v0 + v1) / 2
        
        # Interpolate height from the base altitude map (or current LOD map if using LOD)
        # Source says: "sample of maximum value... is a value of perturbation function"
        # Then: "If more accurate... perform bilinear... from the last level of detail"
        # So we use the sampled h_max as the perturbation for this voxel's contribution
        h_perturbation = h_max
        
        # Check subdivision condition
        # If h > Hmax, stop. But h_max IS the Hmax for this level.
        # The source says: "If level of detail not last level, then calculated height h is compared 
        # to value of height of given level Hmax, and if h > Hmax, then voxel subdivision stops."
        # This logic seems to imply checking if the voxel contains significant terrain features.
        # If the voxel is "empty" or below threshold, stop.
        # Assuming h_perturbation represents the max height in this voxel.
        # If we are not at the leaf level, we might subdivide further if needed.
        # However, the source implies the LOD map *is* the max height.
        # Let's assume we always subdivide until the final LOD level to get precise interpolation.
        
        if lod_idx > 0:
            # Subdivide
            p_mid = (p_near + p_far) / 2.0
            
            # Push children to stack (order may vary)
            stack.append((p_mid, p_far, lod_idx - 1))
            stack.append((p_near, p_mid, lod_idx - 1))
        else:
            # Leaf level: Calculate final points
            # Interpolate precise height at the endpoints or center
            h_final = bilinear_interpolate(u_center, v_center, altitude_map)
            
            # Project center to base surface
            # G(v) gives the projection on the base surface
            proj_point = transform_matrix_g * homogeneous(p_center) # p_center is (p0+p1)/2
            
            # The source equation: F = G(v) + n * h(G(v))
            # G(v) here refers to the 2D coordinate mapped back? 
            # Actually, G is 3D->2D. The equation F = G(v) + n*h(G(v)) is slightly ambiguous in notation.
            # Re-reading S1: "F = G(v) + n * h(G(v))". G(v) is likely the point on the base surface corresponding to v.
            # If base surface is a plane, G(v) is the orthogonal projection of v onto the plane.
            
            # Let's assume G_inv maps (u,v) back to the plane.
            # For a plane with normal n and point p0_plane:
            # proj = v - dot(v - p0_plane, n) * n
            
            # Calculate final point
            # We need the 3D point on the base surface corresponding to the UV coordinates
            # This requires the inverse of the projection part of G.
            # Since G includes T and M, and T swaps y/z, the inverse involves swapping back.
            
            # Simplified: The perturbation is along the normal n.
            # Base point B is the projection of the voxel point onto the base plane.
            # Final point F = B + h * n
            
            # Calculate B (projection of p_center onto base plane)
            # Assuming base plane passes through origin for simplicity, or defined by n and a point
            # Let's assume base plane is defined by normal n and a reference point P_ref
            # B = p_center - dot(p_center - P_ref, n) * n
            
            # For the purpose of this article, we define the base surface projection function
            base_point = project_onto_base_surface(p_center, base_normal)
            
            final_point = base_point + h_final * base_normal
            points.append(final_point)

    return points

def project_onto_base_surface(point, normal):
    # Assuming base plane passes through origin for this implementation
    # If not, a reference point is needed. Source doesn't specify reference point, 
    # but implies a simple surface like a plane.
    # Projection of p onto plane with normal n through origin:
    # p_proj = p - (p . n) * n
    dot_product = dot(point, normal)
    return point - dot_product * normal

def bilinear_interpolate(u, v, table):
    m_u = floor(u)
    m_v = floor(v)
    
    # Handle boundaries if necessary (clamp or wrap)
    # Source doesn't specify boundary handling beyond "outside domain = 0"
    
    u_frac = u - m_u
    v_frac = v - m_v
    
    # Get table values
    # Note: table indices must be within bounds
    val_00 = table[m_u, m_v]
    val_10 = table[m_u + 1, m_v]
    val_01 = table[m_u, m_v + 1]
    val_11 = table[m_u + 1, m_v + 1]
    
    f0 = (1 - u_frac) * val_00 + u_frac * val_10
    f1 = (1 - u_frac) * val_01 + u_frac * val_11
    
    return f0 + (f1 - f0) * v_frac

def homogeneous(point):
    return [point.x, point.y, point.z, 1]
```

**Edge Cases and Pitfalls**
*   **Out-of-Bounds:** If the projected coordinate vector $\mathbf{v}$ falls outside the perturbation domain $D_P$, the perturbation term is zero, and the surface defaults to the base surface [S1, S2]. Implementers must clamp or check bounds before accessing the altitude map table.
*   **Precision:** The method relies on floating-point arithmetic for matrix transformations and interpolation. Precision issues may arise at very large scales or very small height differences [G].
*   **LOD Selection:** The LOD level is determined by the projection size $L_p$. Incorrect calculation of $L_p$ or the inequality thresholds can lead to visual artifacts or incorrect detail levels [S2].

## Operating characteristics
*   **Time Complexity:** The computation time for terrain generation is practically independent of the height map resolution, as the LOD mechanism selects the appropriate resolution dynamically [S2].
*   **Memory Complexity:** Requires storage for the altitude map and its LOD hierarchy. The LOD hierarchy reduces the data size significantly at higher levels [S2].
*   **Performance:**
    *   On Intel Core2 CPU E8400 3.0 GHz: 802.65 ms (128x128), 2033.6 ms (512x512), 3275.6 ms (1024x1024) [S2].
    *   On NVIDIA 9800 GT GPU: 177.03 ms (128x128), 368.9 ms (512x512), 680.46 ms (1024x1024) [S2].
    *   On NVIDIA 470 GTX GPU: 21.09 ms (128x128), 42.51 ms (512x512), 72.35 ms (1024x1024) [S2].
    *   The GPU implementation is on average ten times faster than the CPU-only version [S2].
*   **Strengths:**
    *   Avoids massive vertex/triangle counts associated with polygonal models [S2].
    *   Eliminates cracking problems at LOD boundaries common in voxel-based mesh extraction (e.g., Marching Cubes) [S2].
    *   Low depth complexity (near 1) and no need for hidden polygon removal or pyramid clipping [S2].
    *   Irregular grids can reduce triangle count by a factor of ~64 compared to regular grids for the same error constraints [S2].
*   **Limitations:**
    *   Requires precomputation of LOD maps [S2].
    *   Limited to terrains that can be reasonably approximated by a base surface plus a perturbation (e.g., volcanoes, hills) [S1, S2].

## Background
The method was introduced in a 2015 paper titled "Three-Dimensional Modeling of the Volcano Bandai Using Shape Texture" [S2]. It builds on the concept of "simple surfaces" and "freeform surfaces" defined by scalar fields [S1]. The approach was developed to address the difficulties of rendering complex terrain features (like ridges and canyons) at interactive rates without the computational overhead of traditional polygonal pipelines [S2]. It was tested on the Volcano Bandai model [S1, S2].

## Sources
[S1] GPU-based volume-oriented rendering of functionally defined objects, https://doi.org/10.21279/1454-864x-18-i2-005, fetched 2026-10-26
[S2] Three-Dimensional Modeling of the Volcano Bandai Using Shape Texture, https://doi.org/10.18488/journal.63/2015.3.2/63.2.21.29, fetched 2026-10-26
