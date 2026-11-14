# Bounded 3D Distance Texture Ray Marching

Also called: 3D Texture Distance Function with Boundary Shell, Distance function for a cube sized 3D texture

## Overview
Bounded 3D Distance Texture Ray Marching is a technique for rendering implicit surfaces defined by a 3D distance texture, specifically addressing the problem of invalid distance values at the texture boundaries [S1, S2, unofficial]. It employs two nested bounding boxes (an inner "red" box and an outer "green" box) to distinguish between valid texture data and a boundary shell where samples may be unreliable [S1, S2, unofficial]. Unlike standard texture lookups that might return erroneous distances near edges, this method returns the distance to the outer boundary for points in the invalid shell, ensuring the ray marcher steps safely without tunneling into undefined data [S1, S2, unofficial]. It enables the inclusion of pre-computed distance textures (e.g., the Stanford Bunny) into scenes using standard SDF composition operators [S1, S2, unofficial].

## How it works
**Inputs and Outputs:**
*   **Input:** A query point $\mathbf{x}$ in world space, a 3D distance texture `texture`, and parameters `textureBoundary` and `textureShell`.
*   **Output:** A signed distance value $res$ representing the distance to the implicit surface or the texture boundary.

**Data Structures:**
*   **3D Distance Texture:** A voxel grid storing distance values. In the reference implementation, dimensions are $192 \times 192 \times 192$ [S1, S2, unofficial].
*   **textureBoundary:** A vector defining the half-dimensions of the inner valid region. Value: $(0.5, 0.5, 0.5)$ [S2, unofficial].
*   **textureShell:** A vector defining the thickness of the boundary region where data is considered invalid. Value: $(0.05, 0.05, 0.05)$ [S2, unofficial].

**Algorithm Steps:**
1.  **Calculate Distance to Outer Boundary:** Compute the distance from $\mathbf{x}$ to the inner valid region box. This assumes $\mathbf{x}$ is outside the valid region initially.
    $$ \mathbf{d_i} = |\mathbf{x}| - \text{textureBoundary} + \text{textureShell} $$
    [S2, unofficial]
2.  **Compute Outer Distance (Fallback):** Calculate the norm of the adjusted distance vector. This value is used if the point is outside the valid texture space or in the invalid shell.
    $$ res = \| \max(\mathbf{d_i} + \text{textureShell}, 0) \| $$
    [S2, unofficial]
    *Note: The source uses $\|\cdot\|$ notation for the norm [S2, unofficial].*
3.  **Check Valid Region:** Determine if $\mathbf{x}$ is inside the inner valid region by checking the components of $\mathbf{d_i}$.
    $$ c = \max(d_{ix}, \max(d_{iy}, d_{iz})) $$
    [S2, unofficial]
4.  **Lookup if Inside:** If $c < 0$, the point is inside the valid region.
    *   Adjust coordinates to align with texture space:
        $$ \mathbf{x}_{adj} = \mathbf{x} + \text{textureBoundary} $$
        [S2, unofficial]
    *   Perform texture lookup:
        $$ res = \text{lookupTexture}(\text{texture}, \mathbf{x}_{adj}) $$
        [S2, unofficial]
5.  **Return Result:** Return $res$. If the point was outside the valid region (or in the shell), $res$ holds the distance to the boundary calculated in Step 2. If inside, it holds the texture value [S1, S2, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def bounded_3d_distance_texture_ray_marching(x, texture, texture_boundary, texture_shell):
    """
    Computes the signed distance for a point x using a bounded 3D distance texture.
    
    Args:
        x (vec3): Query point in world space.
        texture (3D Texture): The distance field texture.
        texture_boundary (vec3): Half-dimensions of the inner valid region. Default: (0.5, 0.5, 0.5).
        texture_shell (vec3): Thickness of the invalid boundary shell. Default: (0.05, 0.05, 0.05).
        
    Returns:
        float: Signed distance to the surface or boundary.
    """
    # Step 1: Calculate distance to the inner valid region boundary, expanded by shell
    # di represents the distance to the edge of the valid region, offset by the shell
    di = abs(x) - texture_boundary + texture_shell
    
    # Step 2: Compute fallback distance (distance to the outer boundary of the valid region)
    # This is used for points outside the valid region or in the invalid shell
    # max(di + texture_shell, 0) ensures we only consider positive distances
    # The norm ||...|| is computed component-wise max then Euclidean norm? 
    # Source formula: res = || max(di + textureShell, 0) ||
    # Interpreting ||v|| as Euclidean norm of vector v:
    adjusted_di = max(di + texture_shell, 0.0)
    res = sqrt(adjusted_di.x**2 + adjusted_di.y**2 + adjusted_di.z**2)
    
    # Step 3: Check if point is inside the inner valid region
    # c is the maximum component of di. If c < 0, all components are negative, meaning inside.
    c = max(di.x, max(di.y, di.z))
    
    # Step 4: If inside, perform texture lookup
    if c < 0:
        # Adjust coordinates to map to texture space [0,1] or similar
        x_adj = x + texture_boundary
        # Lookup distance from texture
        res = lookup_texture(texture, x_adj)
        
    # Step 5: Return result
    return res
```

**Parameters and Typical Values:**
*   `textureBoundary`: $(0.5, 0.5, 0.5)$ [S2, unofficial].
*   `textureShell`: $(0.05, 0.05, 0.05)$ [S2, unofficial].
*   Texture Dimensions: $192 \times 192 \times 192$ with an inner valid region of $128 \times 128 \times 128$ [S1, S2, unofficial].

**Edge Cases and Pitfalls:**
*   **Invalid Boundary Data:** Texture samples in the region between the inner valid region and the outer texture edge (the "green box" boundary) may contain invalid distance values [S1, S2, unofficial]. The algorithm remedies this by returning the distance to the outer boundary (red box) for these points, preventing the ray marcher from entering invalid data [S1, S2, unofficial].
*   **Scale Mismatch:** The distance returned by the texture lookup must be in accordance with the scale of the scene. If not, an appropriate bound must be applied [S1, S2, unofficial].
*   **Normal Estimation:** When estimating normals (e.g., using Algorithm 3.1 from the source), the epsilon value must be adjusted to account for the voxel spacing in the texture. If the value is too small, the representation may appear "grainy" [S1, S2, unofficial].
*   **Mesh Holes:** For accurate distance field generation from meshes (like the Stanford Bunny), holes in the mesh must be closed before conversion to a 3D texture [S2, unofficial].

## Subtopics

### [Ambient Occlusion Specific Distance Function](4320-ambient-occlusion-specific-distance-function.md)
A variant that returns a constant step size $\Delta t$ for points outside the texture volume to mitigate AO artifacts caused by the bounding shell.

## Operating characteristics
*   **Time Complexity:** $O(1)$ per query, involving basic vector arithmetic and one texture lookup.
*   **Memory Complexity:** Depends on the 3D texture size. For a $192^3$ texture, this is significant compared to procedural SDFs [S1, S2, unofficial].
*   **Strengths:** Allows the use of pre-computed distance fields from meshes (e.g., Stanford Bunny) in ray marching scenes [S1, S2, unofficial]. Handles boundary conditions robustly to prevent tunneling [S1, S2, unofficial].
*   **Limitations:** Requires storing the 3D texture, which consumes more memory than procedural SDFs. The `textureShell` parameter is heuristic and determined by trial and error, potentially varying between textures [S1, S2, unofficial].
*   **When to Use:** When rendering complex, pre-computed distance fields from meshes where procedural SDFs are insufficient or impractical [S1, S2, unofficial].
*   **When Not to Use:** When memory is constrained or when the scene geometry is simple enough for analytical SDFs [S1, S2, unofficial].

## Background
This method originates from research on GPU ray marching for ambient occlusion and distance fields [S1, S2, unofficial]. It was developed to integrate pre-computed 3D distance textures (such as the Stanford Bunny) into ray marching pipelines, addressing the specific challenge of invalid distance values at the texture boundaries [S1, S2, unofficial]. The technique is described in the context of the GEL (GEometry and Linear algebra) framework [S2, unofficial].

## Sources
[S1] GPU Ray Marching of Distance Fields, https://www2.compute.dtu.dk/pubdb/edoc/imm6392.pdf, fetched 2026-10-27 (unofficial)
[S2] GPU Ray Marching for Ambient Occlusion, https://www.scribd.com/document/520755804/imm6392, fetched 2026-10-28 (unofficial)
