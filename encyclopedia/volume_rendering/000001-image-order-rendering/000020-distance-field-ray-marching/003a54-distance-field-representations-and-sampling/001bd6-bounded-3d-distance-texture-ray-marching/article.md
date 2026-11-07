# Bounded 3D Distance Texture Ray Marching

Also called: 3D Texture Distance Function with Boundary Shell, Distance function for a cube sized 3D texture

## Overview
Bounded 3D Distance Texture Ray Marching renders implicit surfaces defined by a pre-computed 3D signed distance texture by combining direct texture lookups with geometric bounding box checks to handle boundary conditions [S1, unofficial][S2, unofficial]. It solves the problem of invalid or noisy distance values at the edges of a finite texture volume by defining an inner "green" region for safe lookups and an outer bounding box that acts as a hard stop for the ray marcher, preventing the algorithm from sampling invalid data [S1, unofficial][S2, unofficial]. Unlike primitive SDFs, this method allows complex mesh-derived data (e.g., the Stanford Bunny) to be integrated into ray marching scenes using standard boolean operators like union [S1, unofficial][S2, unofficial].

## How it works
**Inputs and Outputs**
*   **Input:** A query point $x$ in world space; a 3D texture `texture` containing signed distance values; parameters `textureBoundary` and `textureShell` [S2, unofficial].
*   **Output:** A scalar distance value $res$ representing the distance to the nearest surface or the boundary of the valid texture region [S1, unofficial][S2, unofficial].

**Data Structures**
*   **3D Texture:** A voxel grid storing signed distance values. The source uses a $192 \times 192 \times 192$ texture with an inner valid region of $128 \times 128 \times 128$ [S1, unofficial][S2, unofficial].
*   **Parameters:**
    *   `textureBoundary`: Half-dimensions of the inner valid region. Value: $(0.5, 0.5, 0.5)$ [S2, unofficial].
    *   `textureShell`: Thickness of the boundary region surrounding the inner valid region. Value: $(0.05, 0.05, 0.05)$ [S2, unofficial].

**Algorithm Steps**
The distance function determines if the query point $x$ is inside the valid texture region or outside it.

1.  **Calculate Distance to Inner Boundary:** Compute the vector $d_i$ representing the distance from $x$ to the inner valid region box, adjusted by the shell thickness.
    $$d_i = |x| - \text{textureBoundary} + \text{textureShell}$$
    Here, $|x|$ denotes the component-wise absolute value of $x$ [S2, unofficial].

2.  **Calculate Distance to Outer Boundary:** If the point is outside the inner region, the distance to the outer bounding box (referred to as the "red box" in the source figures, representing the inner valid region boundary) is calculated. This serves as the SDF value for points outside the texture volume, ensuring the ray marcher stops at the texture boundary [S1, unofficial][S2, unofficial].
    $$res = \| \max(d_i + \text{textureShell}, 0) \|$$
    The notation $\| \cdot \|$ refers to the Euclidean norm [S2, unofficial]. The term $\max(d_i + \text{textureShell}, 0)$ ensures that if the point is inside the shell region, the distance contribution is zeroed out before taking the norm [S2, unofficial].

3.  **Check for Inner Region Entry:** Determine if the point $x$ is inside the inner valid region (the "green box").
    $$c = \max(d_{ix}, \max(d_{iy}, d_{iz}))$$
    If $c < 0$, the point is inside the inner valid region [S2, unofficial].

4.  **Lookup Distance:** If the point is inside the inner region ($c < 0$):
    *   Adjust the point coordinates to align with the texture coordinate space:
        $$x = x + \text{textureBoundary}$$
    *   Perform the texture lookup:
        $$res = \text{lookupTexture}(\text{texture}, x)$$
    *   Return $res$ [S2, unofficial].

5.  **Return Outer Distance:** If the point is not inside the inner region (i.e., $c \ge 0$), return the distance to the outer bounding box calculated in Step 2 [S1, unofficial][S2, unofficial].

**Pseudocode**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def bounded_3d_texture_sdf(x, texture, texture_boundary=(0.5, 0.5, 0.5), texture_shell=(0.05, 0.05, 0.05)):
    # Step 1: Compute di = |x| - textureBoundary + textureShell (component-wise)
    abs_x = (abs(x[0]), abs(x[1]), abs(x[2]))
    di = (
        abs_x[0] - texture_boundary[0] + texture_shell[0],
        abs_x[1] - texture_boundary[1] + texture_shell[1],
        abs_x[2] - texture_boundary[2] + texture_shell[2]
    )
    
    # Step 2: Compute fallback distance res = ||max(di + textureShell, 0)||
    adjusted_di = (
        max(di[0] + texture_shell[0], 0.0),
        max(di[1] + texture_shell[1], 0.0),
        max(di[2] + texture_shell[2], 0.0)
    )
    res = (adjusted_di[0]**2 + adjusted_di[1]**2 + adjusted_di[2]**2)**0.5
    
    # Step 3: Check inner region entry c = max(dix, max(diy, diz))
    c = max(di[0], max(di[1], di[2]))
    
    # Step 4: If inside inner region (c < 0), lookup texture
    if c < 0:
        x_adj = (x[0] + texture_boundary[0], x[1] + texture_boundary[1], x[2] + texture_boundary[2])
        # Standard 3D texture fetch (nearest-neighbor sampling)
        res = texture[int(x_adj[0] * texture.shape[0]), int(x_adj[1] * texture.shape[1]), int(x_adj[2] * texture.shape[2])]
        
    # Step 5: Return result
    return res
```

**Edge Cases and Pitfalls**
*   **Invalid Boundary Data:** Texture samples outside the green box but inside the blue box (the boundary region) may contain invalid distance values [S1, unofficial][S2, unofficial]. The algorithm remedies this by disregarding that region for lookups and using the red box distance instead [S1, unofficial][S2, unofficial].
*   **Scale Mismatch:** The distance returned by the texture lookup must be in accordance with the scale of the scene; otherwise, an appropriate bound must be applied [S1, unofficial].
*   **Normal Estimation:** When estimating normals (e.g., via finite differences), the epsilon value must be adjusted to account for the voxel spacing in the texture. If the value is too small, the representation may appear "grainy" [S1, unofficial][S2, unofficial].
*   **Mesh Holes:** For accurate distance field generation from meshes (e.g., Stanford Bunny), holes in the mesh must be closed before conversion to a 3D distance texture [S2, unofficial].

## Subtopics

### [Ambient Occlusion Specific Distance Function](4320-ambient-occlusion-specific-distance-function.md)
A variant designed to mitigate artifacts caused by the bounding shell of the texture volume contributing to ambient occlusion calculations. It adjusts the distance function to prevent the outer boundary from incorrectly occluding the surface.

## Operating characteristics
*   **Time Complexity:** The SDF evaluation is $O(1)$ per query, involving a few arithmetic operations and one texture lookup [S2, unofficial].
*   **Memory Complexity:** Depends on the resolution of the 3D texture. The source example uses a $192^3$ texture [S1, unofficial][S2, unofficial].
*   **Strengths:** Allows complex, mesh-derived geometry to be used in ray marching; handles boundary conditions robustly by falling back to a geometric bounding box; supports boolean operations (union, intersection) [S1, unofficial][S2, unofficial].
*   **Limitations:** Requires pre-computation of the 3D distance texture; texture resolution limits detail; boundary shell thickness (`textureShell`) requires tuning per texture [S1, unofficial][S2, unofficial].
*   **When to Use:** When rendering complex static geometry defined by a mesh that has been converted to a distance field, especially when integrating it into a larger ray marching scene [S1, unofficial][S2, unofficial].

## Background
The method originates from research on GPU ray marching for ambient occlusion, specifically using 3D textures as implicit surfaces [S1, unofficial][S2, unofficial]. It was applied to render the Stanford Bunny model, which required closing mesh holes before conversion to a 3D distance texture using the GEL framework [S2, unofficial]. The approach enables the inclusion of texture-based representations into scenes using standard SDF boolean operators [S1, unofficial][S2, unofficial].

## Sources
[S1] GPU Ray Marching of Distance Fields, https://www2.compute.dtu.dk/pubdb/edoc/imm6392.pdf, fetched 2026-10-27 (unofficial)
[S2] GPU Ray Marching for Ambient Occlusion, https://www.scribd.com/document/520755804/imm6392, fetched 2026-10-28 (unofficial)
