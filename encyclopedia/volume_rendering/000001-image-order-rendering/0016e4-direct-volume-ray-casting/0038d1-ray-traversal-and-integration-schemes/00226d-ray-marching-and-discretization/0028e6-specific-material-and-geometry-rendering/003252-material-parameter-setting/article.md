# Material Parameter Setting

Also called: Material Format, Material object fields, vpSetMaterial

## Overview
Material Parameter Setting defines the optical properties and texture mappings applied to volume voxels during ray casting [S1, S2]. It solves the problem of decoupling physical density data from visual appearance rules by allowing surface properties to be determined based on geometric orientation relative to the viewer [S2]. It differs from sibling methods by providing a structured object layout for physically-based rendering parameters alongside a legacy API for classical shading coefficients [S1, S2].

## How it works
**Inputs:**
*   Material identifier (index or constant) [S1, S2].
*   Optical properties: Albedo (RGB), Metalness (scalar), Roughness (scalar), Base Reflectivity $F_0$ (RGB) [S1].
*   Texture descriptors (integers mapping to texture units, $-1$ if unused) [S1].
*   Projection mode (Planar, Triplanar) [S1].
*   Normal strength (scalar) [S1].
*   Surface side selection (Exterior, Interior, Both) [S2].

**Outputs:**
*   A configured Material object or state entry used by the ray marching loop to compute shading [S1, S2].

**Data Structures:**
The Material object contains the following fields [S1]:
*   `albedo`: Linear float RGB.
*   `metalness`: Float in $[0, 1]$.
*   `roughness`: Float in $[0, 1]$.
*   `F0`: Linear float RGB (base reflectivity).
*   `triplar`: Boolean (XZ planar vs. triplanar projection).
*   `normalStrength`: Linear float.
*   `albedoTex`: Texture descriptor (int, $-1$ if unused).
*   `normalTex`: Texture descriptor (int, $-1$ if unused).
*   `roughnessTex`: Texture descriptor (int, $-1$ if unused).
*   `metalnessTex`: Texture descriptor (int, $-1$ if unused).
*   `heightmapTex`: Texture descriptor (int, $-1$ if unused, for geometric displacement).
*   `ambietTex`: Texture descriptor (int, $-1$ if unused, for ambient occlusion).

**Numbered Steps:**
1. **PBR Initialization:** Create a Material object [S1]. Set `albedo`, `metalness`, `roughness`, and `F0` from arguments [S1]. Set all texture descriptors to $-1$, projection mode to planar, and `normalStrength` to $1.0$ [S1]. If $F_0$ is omitted, use $[0.04, 0.04, 0.04]$ [S1].
2. **Legacy Initialization:** Call `vpSetMaterial` with the material index, property type, surface side, and color/exponent values [S2]. The renderer updates the internal state for the specified material index [S2].
3. **Surface Side Determination:** Compute the volume gradient $\nabla V$ at the sample point. Compute the dot product with the view direction $\vec{v}$: $d = \nabla V \cdot \vec{v}$ [S2]. If `surface_side` is `VP_EXTERIOR`, apply material if $d > 0$ (gradient points towards viewer) [S2]. If `VP_INTERIOR`, apply if $d < 0$ (gradient points away from viewer) [S2]. If `VP_BOTH_SIDES`, apply regardless of sign [S2]. If `VP_REVERSE_SURFACE_SIDES` is enabled, invert the exterior/interior logic [S2].
4. **Shading Application:** Sample textures if descriptors are $\ge 0$. Combine base properties with sampled values. Compute final color using the selected shading model [S1, S2].

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
# Constants for legacy API
VP_MATERIAL0, VP_MATERIAL1, VP_MATERIAL2, VP_MATERIAL3, VP_MATERIAL4, VP_MATERIAL5 = 0, 1, 2, 3, 4, 5
VP_AMBIENT, VP_DIFFUSE, VP_SPECULAR, VP_SHINYNESS = 0, 1, 2, 3
VP_EXTERIOR, VP_INTERIOR, VP_BOTH_SIDES = 0, 1, 2
VP_REVERSE_SURFACE_SIDES = True

class Material:
    def __init__(self, albedo, metalness, roughness, F0):
        self.albedo = albedo
        self.metalness = metalness
        self.roughness = roughness
        self.F0 = F0
        self.triplar = False
        self.normalStrength = 1.0
        self.albedoTex = -1
        self.normalTex = -1
        self.roughnessTex = -1
        self.metalnessTex = -1
        self.heightmapTex = -1
        self.ambietTex = -1

def set_material_pbr(albedo, metalness, roughness, F0=None):
    if F0 is None:
        F0 = [0.04, 0.04, 0.04]
    return Material(albedo, metalness, roughness, F0)

def set_material_legacy(vpc, material_num, property, surface_side, r, g, b):
    if property == VP_SHINYNESS:
        vpc.material_state[material_num].shininess = r
    elif property in [VP_AMBIENT, VP_DIFFUSE, VP_SPECULAR]:
        vpc.material_state[material_num].set_color(property, r, g, b)
    vpc.material_state[material_num].surface_side = surface_side

def evaluate_surface_side(gradient, view_direction, surface_side, reverse_sides=False):
    dot_product = sum(g * v for g, v in zip(gradient, view_direction))
    is_exterior = dot_product > 0
    is_interior = dot_product < 0
    if reverse_sides:
        is_exterior = not is_exterior
        is_interior = not is_interior
    if surface_side == VP_EXTERIOR:
        return is_exterior
    elif surface_side == VP_INTERIOR:
        return is_interior
    else:
        return True
```

**Edge Cases and Pitfalls:**
*   **Precision:** Albedo and $F_0$ are linear float RGB; ensure gamma correction is handled if input is sRGB [G].
*   **Boundaries:** Texture descriptors must be $-1$ if unused to prevent invalid memory access in device code [S1].
*   **Degenerate Input:** Metalness and Roughness must be clamped to $[0, 1]$ [G].
*   **Gradient Noise:** Noisy gradients can cause flickering on Exterior/Interior boundaries; smoothing may be required [G].

## Subtopics

### PBR Material Object
Uses albedo, metalness, roughness, and $F_0$. Best for physically accurate rendering. Supports texture mapping for all parameters. [Link](4395-specific-material-and-geometry-rendering.md)

### Legacy Phong Material
Uses ambient, diffuse, specular, and shinyness coefficients. Simpler, faster, but less physically accurate. [Link](4395-specific-material-and-geometry-rendering.md)

### Volume Material Setup
Extracts dimensions `dimX, dimY, dimZ` from `d.meta.dim` and initializes intensity range clamping values `p1=0f`, `p99=1f`, updating them if `d.tf.type == "continuous"` and `d.meta.intensity_range` exists with length 2 [S3, unofficial]. Constructs an affine matrix from `d.meta.affine` and computes its inverse `invAffine` [S3, unofficial]. Selects label texture `texLbl = volumeTexLabels ?? d.labelTexture ?? BlackTex3D` and weight texture `texW = volumeTexWeights ?? d.weightTexture ?? null`, binding `_VolumeTexWeights` to `texW` or `BlackTex3D` and setting `_HasWeights = 1` if `texW != null`, else `0` [S3, unofficial]. Binds `_TFTex` to `tfTexCurrent`, sets `_IsLabelMap = 1` if `d.tf.type == "labelmap"`, else `0`, and sets `_P1`, `_P99`, `_Dim` (Vector4(dimX, dimY, dimZ, 1f)), `_Affine`, `_InvAffine` before assigning `volumeMaterial` to `MeshRenderer.sharedMaterial` [S3, unofficial]. Parameters include `_IsLabelMap` (int flag 0 or 1) and `_P1`, `_P99` (floats for continuous TF intensity range clamping) [S3, unofficial].

## Operating characteristics
*   **Time Complexity:** $O(1)$ per material setup. Shading evaluation is $O(1)$ per sample.
*   **Memory Complexity:** $O(1)$ per material object. Texture memory depends on resolution.
*   **Measured Performance:** Not given in the sources.
*   **Strengths:** Decouples geometry from appearance. Supports complex surface interactions via PBR.
*   **Limitations:** Legacy API is limited to 6 materials. PBR requires more complex shader logic.
*   **When to Use:** Use PBR for realistic materials. Use Legacy for simple visualization or performance-critical applications with limited material slots.

## Background
The Material Format describes the structure and implementation of material objects for raymarching distance fields [S1]. The legacy `vpSetMaterial` function originates from the VolPack library [S2].

## Sources
[S1] Raymarching Distance Fields with CUDA, https://doi.org/10.3390/electronics10222730, fetched 2026-10-26
[S2] VolPack User's Guide, http://www-graphics.stanford.edu/software/volpack/vp_userguide.html, fetched 2026-10-28
[S3] guillaume-schneider/vrdf-sdk, https://github.com/guillaume-schneider/vrdf-sdk, fetched 2026-10-26 (unofficial)
