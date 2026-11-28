# Two-sided Lighting

Also called: Two-sided Directional Lighting

## Overview
Two-sided lighting is a shading mode for surface displays that illuminates both the front and back faces of a surface geometry, rather than restricting illumination to the exterior [S1, S2]. It differs from standard one-sided lighting by treating the specified light direction $\mathbf{l}$ and its opposite $-\mathbf{l}$ as active sources, allowing the interior or back-facing surfaces of a volume boundary to receive direct illumination [S2].

## How it works
**Inputs and Outputs:**
*   **Input:** Surface geometry defined by a volume threshold or mesh; light direction vector $\mathbf{l}$; light color $\mathbf{c}_{light}$; material properties (ambient $\mathbf{c}_{amb}$, diffuse $\mathbf{c}_{diff}$); viewing direction $\mathbf{v}$ (implicit in brightness calculation).
*   **Output:** Final pixel color for surface points, combining ambient and diffuse contributions from both light directions.

**Data Structures:**
The implementation relies on a rendering context storing light states. Key constants include `VP_LIGHT0` through `VP_LIGHT5` for light identifiers, and `VP_LIGHT_BOTH_SIDES` as a flag for the enable state [S2]. Material properties are stored per material index (`VP_MATERIAL0`–`VP_MATERIAL5`) with separate coefficients for ambient and diffuse reflection [S2].

**Steps:**
1.  **Configure Light Color:** Set the RGB components of the light color $\mathbf{c}_{light}$ using `vpSetLight` with property `VP_COLOR`. Values must be in the range $[0.0, 1.0]$ [S2].
2.  **Configure Light Direction:** Set the direction vector $\mathbf{l} = (l_x, l_y, l_z)$ using `vpSetLight` with property `VP_DIRECTION`. This vector is transformed by the current modeling matrix before storage [S2].
3.  **Enable Light:** Call `vpEnable` with the light identifier (e.g., `VP_LIGHT1`) and a value of 1 to activate it. Light 0 is enabled by default; others are disabled [S2].
4.  **Enable Two-Sided Mode:** Call `vpEnable` with the light identifier and the `VP_LIGHT_BOTH_SIDES` option. This instructs the renderer to treat the light as bidirectional [S2].
5.  **Evaluate Illumination:** For each surface point $\mathbf{p}$ with normal $\mathbf{n}$:
    *   Calculate the angle between the normal $\mathbf{n}$ and the line of sight. Brightness is maximal when $\mathbf{n}$ is parallel to the line of sight [S1].
    *   Compute diffuse contribution from both $\mathbf{l}$ and $-\mathbf{l}$.
    *   Combine with ambient term defined by material parameters [S2].

**Formulas:**
The brightness varies according to the angle between the surface point normal and the line of sight [S1]. While the exact shading equation is not explicitly provided in the sources, the diffuse component $L_{diff}$ for a two-sided light typically follows a modified Lambertian model where the dot product is taken with both $\mathbf{l}$ and $-\mathbf{l}$:
$$ L_{diff} = \mathbf{c}_{diff} \cdot \left( \max(0, \mathbf{n} \cdot \mathbf{l}) + \max(0, \mathbf{n} \cdot (-\mathbf{l})) \right) \cdot \mathbf{c}_{light} $$
The total color is:
$$ \mathbf{C}_{final} = \mathbf{c}_{amb} + L_{diff} $$
where $\mathbf{c}_{amb}$ and $\mathbf{c}_{diff}$ are retrieved from the material settings [S2].

**Parameters:**
*   **Light Direction:** Transformed by the modeling matrix [S2].
*   **Color Range:** $[0.0, 1.0]$ for RGB components [S2].
*   **Light Count:** Supports up to 6 lights (`VP_LIGHT0`–`VP_LIGHT5`) [S2].

**Edge Cases and Pitfalls:**
*   **Normal Definition:** The definition of "outside" depends on the `flipNormals` setting. If `twoSidedLighting` is false, `flipNormals` determines which side is lit. When `twoSidedLighting` is true, both sides are lit regardless, but the normal orientation affects the dot product calculation [S1].
*   **Threshold Sign:** For volume data, the side treated as "outside" for negative thresholds is toward larger/more positive values, and for positive thresholds toward smaller/more negative values, if `flipNormals` is true [S1].

## Operating characteristics
*   **Complexity:** Constant time per surface point relative to the number of lights, as it evaluates two directions per light.
*   **Performance:** Adds minimal overhead compared to one-sided lighting, as it simply evaluates the shading function for the opposite normal direction.
*   **Strengths:** Essential for visualizing the internal structure or back-facing surfaces of translucent or cut-away volume renderings.
*   **Limitations:** Only applies to surface displays (isosurfaces), not direct volume ray marching. Does not handle self-shadowing between front and back faces.

## Background
Two-sided lighting originated in the VolPack library, a volume visualization toolkit developed by Stanford University [S2]. It was designed to provide flexible shading options for surface displays derived from volumetric data, particularly useful when the user needs to see both the exterior and interior boundaries of a segmented region [S1, S2]. The feature was later adopted by visualization tools such as UCSF ChimeraX [S1].

## Sources
[S1] Command: volume, https://www.rbvi.ucsf.edu/chimerax/docs/user/commands/volume.html, fetched 2026-10-27
[S2] VolPack User's Guide, http://www-graphics.stanford.edu/software/volpack/vp_userguide.html, fetched 2026-10-28
