# Bounded 3D Distance Texture Ray Marching

Also called: 3D Texture Distance Function with Boundary Shell, Distance function for a cube sized 3D texture

## Overview
Bounded 3D Distance Texture Ray Marching is a technique for rendering implicit surfaces defined by a 3D distance texture, specifically addressing the problem of invalid or noisy distance values at the texture boundaries [S1, unofficial][S2, unofficial]. It distinguishes itself from standard texture lookups by employing two nested bounding volumes (an inner "red" box and an outer "green" box) to separate valid surface data from a boundary shell where distance values may be unreliable [S1, unofficial]. When a ray is outside the valid region, it returns the distance to the inner box boundary, ensuring the ray marcher stops at the texture volume rather than sampling invalid data [S1, unofficial]. This method builds on [Distance Field Representations and Sampling](8725-distance-field-representations-and-sampling.md) by providing a robust sampling function for cube-sized 3D textures.

## How it works
**Inputs:**
- Ray origin $\mathbf{o}$ and direction $\mathbf{d}$.
- 3D Distance Texture $T$ with dimensions $D_x \times D_y \times D_z$ (e.g., $192 \times 192 \times 192$) [S1, unofficial][S2, unofficial].
- Parameters:
  - `textureBoundary`: Half-dimensions of the inner valid region. Default: $(0.5, 0.5, 0.5)$ [S2, unofficial].
  - `textureShell`: Thickness of the boundary region surrounding the inner valid region. Default: $(0.05, 0.05, 0.05)$ [S2, unofficial].
  - `lookupTexture`: Function to sample $T$ at normalized coordinates $[0,1]^3$.

**Outputs:**
- Distance $t$ to the nearest surface or texture boundary.

**Data Structures:**
- The 3D texture $T$ stores signed distance values. The "inner region" (red box) contains the surface and valid distances. The "boundary region" (between red and green boxes) may contain invalid values. The "outer region" (outside green box) is empty space [S1, unofficial].

**Algorithm:**
The distance function $f(\mathbf{x})$ for a query point $\mathbf{x}$ is computed as follows:

1.  **Compute Distance to Inner Boundary:**
    Calculate the distance from $\mathbf{x}$ to the inner valid region box defined by `textureBoundary`. This assumes $\mathbf{x}$ is outside the inner box.
    $$ \mathbf{di} = |\mathbf{x}| - \text{textureBoundary} + \text{textureShell} $$
    Here, $|\mathbf{x}|$ denotes the component-wise absolute value of $\mathbf{x}$, and operations are component-wise [S2, unofficial].

2.  **Compute Fallback Distance:**
    Calculate the Euclidean norm of the positive part of $\mathbf{di}$. This represents the distance to the "red box" (inner boundary) if the point is outside the valid region.
    $$ \text{res} = \| \max(\mathbf{di} + \text{textureShell}, 0) \| $$
    Note: The source formula uses $\| \max(\mathbf{di} + \text{textureShell}, 0) \|$ for points outside the 3D texture space [S2, unofficial]. *Correction based on source text:* The source states `res = k max(di + textureShell, 0)k` for points outside. However, line 3 defines `di`. Line 4 uses `di`. Let's follow the pseudocode structure strictly.
    
    *Re-evaluating Source [S2, unofficial] Algorithm 4.1:*
    Line 3: `di = |x| - textureBoundary + textureShell`
    Line 4: `res = k max(di + textureShell, 0)k` (for x outside 3D texture space)
    Line 5: `c = max(dix, max(diy, diz))`
    Line 6: `if c < 0 then`
    Line 7: `x = x + textureBoundary`
    Line 8: `res = lookupTexture(texture, x)` (for x in 3D texture space)
    Line 9: `return res`

    There is a slight ambiguity in the source text regarding the exact formula in Line 4 vs Line 3. Line 3 computes `di`. Line 4 computes `res` using `di`. The condition `c < 0` checks if the point is *inside* the inner valid region (shifted by shell). If `c < 0`, the point is inside the valid region, so we lookup. Otherwise, we return `res` (distance to the boundary).

    Let's refine the steps for implementation clarity:

    1.  **Shift and Scale:** The texture is assumed to be centered at the origin with dimensions normalized to $[-1, 1]$ or similar, but `textureBoundary` is $(0.5, 0.5, 0.5)$. The input $\mathbf{x}$ is in world space. The algorithm assumes $\mathbf{x}$ is in a coordinate system where the texture occupies $[-1, 1]^3$ or similar, such that `textureBoundary` represents the half-extent.
    
    2.  **Calculate Component-wise Distance to Inner Box:**
        $$ \mathbf{di} = |\mathbf{x}| - \text{textureBoundary} + \text{textureShell} $$
        This vector $\mathbf{di}$ represents the signed distance components to the inner box expanded by the shell.

    3.  **Check if Inside Valid Region:**
        $$ c = \max(dix, \max(diy, diz)) $$
        If $c < 0$, the point $\mathbf{x}$ is inside the inner valid region (the "red box" defined by `textureBoundary - textureShell`).

    4.  **Lookup or Fallback:**
        - If $c < 0$:
            - Adjust coordinates to align with texture space $[0, 1]^3$:
              $$ \mathbf{x}_{tex} = \mathbf{x} + \text{textureBoundary} $$
              (Assuming $\mathbf{x}$ is in $[-1, 1]^3$ range, adding $0.5$ shifts it to $[-0.5, 0.5]$? No, `textureBoundary` is $(0.5,0.5,0.5)$. If $\mathbf{x}$ is in $[-1,1]$, adding $0.5$ gives $[-0.5, 1.5]$. This implies the texture coordinate mapping might be specific. The source says `x = x + textureBoundary`. We follow this exactly [S2, unofficial]).
            - Sample the texture:
              $$ \text{res} = \text{lookupTexture}(T, \mathbf{x}_{tex}) $$
        - Else ($c \ge 0$):
            - The point is outside the inner valid region. Return the distance to the boundary shell.
            $$ \text{res} = \| \max(\mathbf{di} + \text{textureShell}, 0) \| $$
            *Note on Formula:* The source line 4 is `res = k max(di + textureShell, 0)k`. Since `di` already includes `+ textureShell` in line 3, this effectively computes the norm of `max(di, textureShell)`. However, we must follow the source's explicit formula [S2, unofficial].

    **Pseudocode:**

    ```python
    def bounded_3d_distance_texture_ray_march(x, texture, texture_boundary, texture_shell):
        """
        Computes the distance from point x to the implicit surface in a 3D texture.
        
        Args:
            x: vec3, query point in world space (assumed to be in range [-1, 1]^3 or similar 
               such that texture_boundary=(0.5,0.5,0.5) defines the inner box).
            texture: 3D array, the distance texture.
            texture_boundary: vec3, half-dimensions of the inner valid region. Default (0.5, 0.5, 0.5).
            texture_shell: vec3, thickness of the boundary region. Default (0.05, 0.05, 0.05).
            
        Returns:
            float, distance to surface or boundary.
        """
        # Step 1: Compute di = |x| - textureBoundary + textureShell
        abs_x = abs(x)
        di = abs_x - texture_boundary + texture_shell
        
        # Step 2: Check if inside inner valid region
        # c = max(dix, max(diy, diz))
        c = max(di.x, max(di.y, di.z))
        
        if c < 0:
            # Inside valid region
            # Adjust coordinates: x = x + textureBoundary
            x_tex = x + texture_boundary
            
            # Lookup texture
            # Note: lookupTexture assumes normalized coordinates [0,1] or similar.
            # The source does not specify further normalization, so we assume 
            # x_tex is in the correct range for the texture sampler.
            res = lookup_texture(texture, x_tex)
        else:
            # Outside valid region
            # res = || max(di + textureShell, 0) ||
            # Note: di already has textureShell added. 
            # Source formula: max(di + textureShell, 0)
            shifted_di = di + texture_shell
            positive_di = max(shifted_di, 0.0) # Component-wise max with 0
            res = length(positive_di)
            
        return res
    ```

**Edge Cases and Pitfalls:**
- **Invalid Boundary Data:** Texture samples in the boundary region (between the red and green boxes) may contain invalid distance values. The algorithm remedies this by returning the distance to the red box (inner boundary) for these points, preventing the ray marcher from entering invalid data [S1, unofficial][S2, unofficial].
- **Scale Mismatch:** The distance returned by the texture lookup must be in accordance with the scale of the scene. If not, an appropriate bound must be applied [S1, unofficial].
- **Normal Estimation:** When estimating normals (e.g., using Algorithm 3.1 from the source), the epsilon value must be adjusted to account for the voxel spacing in the texture. If the value is too small, the rendered surface may appear "grainy" [S1, unofficial][S2, unofficial].
- **Mesh Holes:** If the 3D texture is generated from a mesh (e.g., Stanford Bunny), holes in the mesh must be closed before conversion to ensure correct distance values [S2, unofficial].

## Subtopics

### [Ambient Occlusion Specific Distance Function](4320-ambient-occlusion-specific-distance-function.md)
A variant that modifies the boundary handling logic for AO shading. Instead of returning the distance to the red box for points outside the green box, it returns a constant step size $\Delta t$. This prevents the bounding "red box" from contributing to the occlusion result, avoiding visible artifacts on surfaces like the bunny's face. Choose this when rendering AO with 3D distance textures to eliminate boundary artifacts.

## Operating characteristics
- **Time Complexity:** $O(1)$ per distance query. The computation involves a few vector operations and one texture lookup.
- **Memory Complexity:** $O(D_x D_y D_z)$ for the 3D texture storage. For a $192 \times 192 \times 192$ texture, this is significant but manageable on modern GPUs [S1, unofficial][S2, unofficial].
- **Measured Performance:** The method was demonstrated on the Stanford Bunny model with a $192 \times 192 \times 192$ texture and an inner valid region of $128 \times 128 \times 128$ [S1, unofficial][S2, unofficial].
- **Strengths:** Robust handling of texture boundaries; prevents ray marchers from sampling invalid data; allows standard SDF operations (like union) to be applied to textured representations [S1, unofficial].
- **Limitations:** Requires pre-computation of the 3D distance texture; memory intensive for high-resolution textures; boundary shell thickness (`textureShell`) requires tuning per texture [S1, unofficial].
- **When to Use:** When rendering complex implicit surfaces defined by 3D distance textures, especially when boundary artifacts are a concern.
- **When Not to Use:** When memory is constrained, or when the surface is simple enough to be represented by analytical SDFs.

## Background
The method originates from research on GPU ray marching of distance fields, specifically for rendering implicit surfaces represented by 3D textures [S1, unofficial][S2, unofficial]. It was developed to handle the limitations of 3D distance textures, where boundary values are often unreliable. The technique was applied to render the Stanford Bunny model, demonstrating its effectiveness in both standard ray marching and ambient occlusion contexts [S1, unofficial][S2, unofficial]. The work is part of a broader effort to integrate 3D textures into ray marching pipelines, allowing for complex geometry representation without explicit mesh traversal [S1, unofficial].

## Sources
[S1] GPU Ray Marching of Distance Fields, https://www2.compute.dtu.dk/pubdb/edoc/imm6392.pdf, fetched 2026-10-27 (unofficial)
[S2] GPU Ray Marching for Ambient Occlusion, https://www.scribd.com/document/520755804/imm6392, fetched 2026-10-28 (unofficial)
