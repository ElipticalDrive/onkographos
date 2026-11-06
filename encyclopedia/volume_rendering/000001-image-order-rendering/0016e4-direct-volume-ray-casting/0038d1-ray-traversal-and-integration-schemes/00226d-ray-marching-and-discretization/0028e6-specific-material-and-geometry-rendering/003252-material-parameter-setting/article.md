# Material Parameter Setting

Also called: Material Format, Material object fields, vpSetMaterial

## Overview
Material Parameter Setting defines the optical properties (albedo, metalness, roughness, reflectivity) and texture mappings applied to volume voxels during ray casting. It solves the problem of decoupling physical density data from visual appearance rules by allowing surface properties to be determined based on geometric orientation (gradient direction) relative to the viewer. It differs from sibling methods by providing a structured object layout for physically-based rendering (PBR) parameters alongside a legacy API for classical Phong-style shading coefficients.

## How it works
**Inputs:**
*   Material identifier (index or constant).
*   Optical properties: Albedo (RGB), Metalness (scalar), Roughness (scalar), Base Reflectivity $F_0$ (RGB).
*   Texture descriptors (integers mapping to texture units, -1 if unused).
*   Projection mode (Planar, Triplanar).
*   Normal strength (scalar).
*   Surface side selection (Exterior, Interior, Both).

**Outputs:**
*   A configured Material object or state entry used by the ray marching loop to compute shading.

**Data Structures:**
The Material object contains the following fields [S1]:
*   `albedo`: Linear float RGB.
*   `metalness`: Float in $[0, 1]$.
*   `roughness`: Float in $[0, 1]$.
*   `F0`: Linear float RGB (base reflectivity).
*   `triplar`: Boolean (XZ planar vs. triplanar projection).
*   `normalStrength`: Linear float.
*   `albedoTex`: Texture descriptor (int, -1 if unused).
*   `normalTex`: Texture descriptor (int, -1 if unused).
*   `roughnessTex`: Texture descriptor (int, -1 if unused).
*   `metalnessTex`: Texture descriptor (int, -1 if unused).
*   `heightmapTex`: Texture descriptor (int, -1 if unused, for geometric displacement).
*   `ambietTex`: Texture descriptor (int, -1 if unused, for ambient occlusion).

**Numbered Steps:**

1.  **Initialization (PBR Style):**
    *   Create a Material object.
    *   Set `albedo`, `metalness`, `roughness`, and `F0` from arguments.
    *   Set all texture descriptors (`albedoTex`, `normalTex`, etc.) to $-1$.
    *   Set projection mode to planar.
    *   Set `normalStrength` to $1.0$.
    *   [G] Ensure the material is registered in the renderer's material table.

2.  **Initialization (Legacy Style):**
    *   Call `vpSetMaterial` with the material index, property type, surface side, and color/exponent values.
    *   The renderer updates the internal state for the specified material index.

3.  **Surface Side Determination:**
    *   Compute the volume gradient $\nabla V$ at the sample point.
    *   Compute the dot product with the view direction $\vec{v}$: $d = \nabla V \cdot \vec{v}$.
    *   If `surface_side` is `VP_EXTERIOR`: Apply material if $d > 0$ (gradient points towards viewer).
    *   If `surface_side` is `VP_INTERIOR`: Apply material if $d < 0$ (gradient points away from viewer).
    *   If `surface_side` is `VP_BOTH_SIDES`: Apply material regardless of sign.
    *   [G] If `VP_REVERSE_SURFACE_SIDES` is enabled, invert the logic for Exterior/Interior.

4.  **Shading Application:**
    *   Sample textures if descriptors are $\ge 0$.
    *   Combine base properties with sampled values.
    *   Compute final color using the selected shading model (PBR or Phong).

**Formulas:**
*   Base Reflectivity for non-metal dielectrics: $F_0 = [0.04, 0.04, 0.04]$ [S1].
*   Surface Side Condition:
    $$ \text{Apply} = \begin{cases} 
    (\nabla V \cdot \vec{v} > 0) & \text{if Exterior} \\
    (\nabla V \cdot \vec{v} < 0) & \text{if Interior} \\
    \text{True} & \text{if Both}
    \end{cases} $$

**Parameters:**
*   `material_num`: Identifier for the material. Constants: `VP_MATERIAL0` through `VP_MATERIAL5` [S2].
*   `property`: The property to set. Options: `VP_AMBIENT`, `VP_DIFFUSE`, `VP_SPECULAR`, `VP_SHINYNESS` [S2].
*   `surface_side`: `VP_EXTERIOR`, `VP_INTERIOR`, `VP_BOTH_SIDES` [S2].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def set_material_pbr(albedo, metalness, roughness, F0=None):
    """
    Initialize a Material object using PBR parameters.
    """
    if F0 is None:
        F0 = [0.04, 0.04, 0.04]
    
    material = Material()
    material.albedo = albedo
    material.metalness = metalness
    material.roughness = roughness
    material.F0 = F0
    
    # Set texture descriptors to -1 (unused)
    material.albedoTex = -1
    material.normalTex = -1
    material.roughnessTex = -1
    material.metalnessTex = -1
    material.heightmapTex = -1
    material.ambietTex = -1
    
    # Set default projection and normal strength
    material.triplar = False  # Planar
    material.normalStrength = 1.0
    
    return material

def set_material_legacy(vpc, material_num, property, surface_side, r, g, b):
    """
    Set material properties using the VolPack API style.
    """
    # Implementation detail: Updates internal state of vpc for material_num
    # property is one of: VP_AMBIENT, VP_DIFFUSE, VP_SPECULAR, VP_SHINYNESS
    # For VP_SHINYNESS, g and b are ignored.
    
    if property == VP_SHINYNESS:
        # Set specular exponent
        pass
    elif property in [VP_AMBIENT, VP_DIFFUSE, VP_SPECULAR]:
        # Set RGB coefficients
        pass
        
    # surface_side is VP_EXTERIOR, VP_INTERIOR, or VP_BOTH_SIDES
    pass

def evaluate_surface_side(gradient, view_direction, surface_side, reverse_sides=False):
    """
    Determine if the current voxel should use the material based on gradient orientation.
    """
    dot_product = dot(gradient, view_direction)
    
    is_exterior = dot_product > 0
    is_interior = dot_product < 0
    
    if reverse_sides:
        is_exterior = not is_exterior
        is_interior = not is_interior
        
    if surface_side == VP_EXTERIOR:
        return is_exterior
    elif surface_side == VP_INTERIOR:
        return is_interior
    else: # VP_BOTH_SIDES
        return True
```

**Edge Cases and Pitfalls:**
*   **Precision:** Albedo and $F_0$ are linear float RGB; ensure gamma correction is handled if input is sRGB.
*   **Boundaries:** Texture descriptors must be $-1$ if unused to prevent invalid memory access in device code [S1].
*   **Degenerate Input:** Metalness and Roughness must be clamped to $[0, 1]$.
*   **Gradient Noise:** Noisy gradients can cause flickering on Exterior/Interior boundaries; smoothing may be required [G].

## Subtopics

### PBR Material Object
Uses albedo, metalness, roughness, and $F_0$. Best for physically accurate rendering. Supports texture mapping for all parameters. [Link](4395-specific-material-and-geometry-rendering.md)

### Legacy Phong Material
Uses ambient, diffuse, specular, and shinyness coefficients. Simpler, faster, but less physically accurate. [Link](4395-specific-material-and-geometry-rendering.md)

## Operating characteristics
*   **Time Complexity:** $O(1)$ per material setup. Shading evaluation is $O(1)$ per sample.
*   **Memory Complexity:** $O(1)$ per material object. Texture memory depends on resolution.
*   **Strengths:** Decouples geometry from appearance. Supports complex surface interactions via PBR.
*   **Limitations:** Legacy API is limited to 6 materials. PBR requires more complex shader logic.
*   **When to Use:** Use PBR for realistic materials. Use Legacy for simple visualization or performance-critical applications with limited material slots.

## Background
The Material Format was introduced in the context of CUDA-based raymarching distance fields [S1]. The legacy `vpSetMaterial` function originates from the VolPack library [S2].

## Sources
[S1] Raymarching Distance Fields with CUDA, https://doi.org/10.3390/electronics10222730, fetched 2026-10-26
[S2] VolPack User's Guide, http://www-graphics.stanford.edu/software/volpack/vp_userguide.html, fetched 2026-10-28
