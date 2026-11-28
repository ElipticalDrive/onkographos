# Voxel Number Map with LUT for Translucent Rendering

Also called: VNM method, Voxel Number Map (VNM) with Look-Up-Table (LUT) for Translucent Rendering, Voxel-Number-Map technique

## Overview
This method is a real-time rendering technique for heterogeneous translucent objects containing complex internal structures, such as cavities and multiple distinct media [S1, unofficial]. It solves the problem of accurately computing subsurface scattering in concave or hollow objects, where traditional Translucent Shadow Maps (TSM) fail due to thickness calculation errors [S1, unofficial]. Unlike TSM, it handles concave shapes accurately; unlike Multi-view Depth Maps, it correctly handles objects with internal cavities and supports heterogeneous media [S1, unofficial]. It builds on [Real-time approximations](36-real-time-approximations.md) by precomputing optical thickness in a Look-Up-Table (LUT) and accessing it via a screen-space Voxel-Number-Map (VNM) [S1, unofficial].

## How it works
The method assumes that single-scattering effects are dominant, in-scattering is negligible, and internal reflection/refraction at media interfaces are sufficiently small [S1, unofficial]. It operates in two phases: precomputation and runtime.

### Inputs and Outputs
**Inputs:**
- Triangle mesh model of the translucent object.
- Surface normals for each vertex.
- Extinction coefficients ($\sigma_t$) for each medium layer.
- Light direction ($\vec{\omega}_{in}$) and irradiance impulse $I(\vec{\omega}_{in})$.
- Relative index of refraction ($\eta$).
- Camera view (user's view).
- Light view (for generating VNM and Irradiance Map).

**Outputs:**
- Final radiance $L(\vec{x}_{out}, \vec{\omega}_{out})$ for each pixel.

### Data Structures
1.  **Voxel Grid:** A 3D grid discretizing the object space. The resolution is defined by the user (e.g., $25 \times 25 \times 25$ or $60 \times 60 \times 60$) [S1, unofficial].
2.  **Surface Voxel List:** A list of voxels from the grid that lie on the object's surface. Each surface voxel is assigned a unique integer ID.
3.  **Texture Coordinate Map:** A mapping from each surface voxel ID to the $(u, v)$ texture coordinates of the original object's vertex [S1, unofficial].
4.  **Look-Up-Table (LUT):** A 2D texture storing optical thickness values. The dimensions correspond to the number of surface voxels. A texel at index $(i, j)$ stores the optical thickness $\tau(\vec{x}_i, \vec{x}_j)$ between surface voxel $i$ and surface voxel $j$ for a specific medium [S1, unofficial].
5.  **Voxel-Number-Map (VNM):** A screen-space texture generated from the light's view. Each pixel stores the unique ID of the surface voxel visible at that screen position [S1, unofficial].
6.  **Irradiance Map:** A screen-space texture generated from the light's view. Each pixel stores the computed irradiance $E(\vec{x}_{in})$ for the visible surface point [S1, unofficial].

### Precomputation Steps
1.  **Voxelization:** Voxelize the object structure by sampling the surface of the object into the 3D voxel grid [S1, unofficial].
2.  **Surface Extraction and Numbering:** Extract all voxels that constitute the surface of the object. Assign a unique integer ID to each surface voxel [S1, unofficial].
3.  **Coordinate Mapping:** Map each surface voxel ID to the $(u, v)$ texture coordinates of the corresponding vertex on the original mesh [S1, unofficial].
4.  **Optical Thickness Computation:** For each pair of surface voxels $(\vec{x}_i, \vec{x}_j)$, compute the optical thickness $\tau(\vec{x}_i, \vec{x}_j)$ along the path between them. This is done for each distinct medium present in the object [S1, unofficial].
5.  **LUT Population:** Store the computed optical thickness values into the LUT texture. The LUT is indexed by the IDs of the surface voxels [S1, unofficial].

### Runtime Steps
**First Pass (Light View):**
1.  **Irradiance Computation:** From the light's view, compute the irradiance $E(\vec{x}_{in})$ for each visible surface point. Store the result in the Irradiance Map [S1, unofficial].
    $$ E(\vec{x}_{in}) = F_t(\eta, \vec{\omega}_{in}) |N(\vec{x}_{in}) \cdot \vec{\omega}_{in}| I(\vec{\omega}_{in}) $$
    Where:
    - $F_t(\eta, \vec{\omega}_{in})$ is the Fresnel transmittance at the interface.
    - $N(\vec{x}_{in})$ is the surface normal at the incident point.
    - $\vec{\omega}_{in}$ is the light direction.
    - $I(\vec{\omega}_{in})$ is the irradiance impulse from the light source [S1, unofficial].
2.  **VNM Generation:** Generate the Voxel-Number-Map (VNM) from the light's view. For each pixel in the screen space, store the unique ID of the surface voxel visible at that location [S1, unofficial].

**Second Pass (User View):**
1.  **Radiance Integral Setup:** For each pixel in the user's view, identify the emission point $\vec{x}_{out}$ and the view direction $\vec{\omega}_{out}$ [S1, unofficial].
2.  **Subsurface Reflectance Lookup:** To compute the diffuse subsurface reflectance $R_d(\vec{x}_{in}, \vec{x}_{out})$, the method needs the optical thickness $\tau(\vec{x}_{in}, \vec{x}_{out})$ between the incident point $\vec{x}_{in}$ and the emission point $\vec{x}_{out}$ [S1, unofficial].
    - The incident point $\vec{x}_{in}$ is determined by sampling the Irradiance Map and VNM. The VNM provides the ID of the surface voxel at the incident location.
    - Using the VNM, fetch the texture coordinates $(u, v)$ associated with the incident surface voxel ID.
    - Use these $(u, v)$ coordinates to sample the LUT to retrieve the optical thickness $\tau(\vec{x}_{in}, \vec{x}_{out})$ [S1, unofficial].
    - Compute $R_d$:
      $$ R_d(\vec{x}_{in}, \vec{x}_{out}) = e^{-\tau(\vec{x}_{in}, \vec{x}_{out})} $$
      Where $\tau(\vec{x}_{in}, \vec{x}_{out})$ is the optical thickness:
      $$ \tau(\vec{x}_{in}, \vec{x}_{out}) = \int_{\vec{x}_{in}}^{\vec{x}_{out}} \sigma_t(\vec{x}') d\vec{x}' $$
      Where $\sigma_t(\vec{x}')$ is the extinction coefficient [S1, unofficial].
3.  **Final Radiance Calculation:** Compute the final radiance using a 21-taps sampling filter, identical to the TSM method, to approximate the integral in Equation 1 [S1, unofficial].
    $$ L(\vec{x}_{out}, \vec{\omega}_{out}) = \frac{1}{\pi} \int_{S} F_t(\eta, \vec{\omega}_{out}) E(\vec{x}_{in}) R_d(\vec{x}_{in}, \vec{x}_{out}) d\vec{x}_{in} $$
    Where:
    - $F_t(\eta, \vec{\omega}_{out})$ is the Fresnel transmittance at the exit interface.
    - $\eta$ is the relative index of refraction.
    - $E(\vec{x}_{in})$ is the irradiance fetched from the Irradiance Map.
    - $R_d(\vec{x}_{in}, \vec{x}_{out})$ is the subsurface reflectance fetched from the LUT via the VNM [S1, unofficial].

### Parameters
- **Voxel Resolution:** Determines the granularity of the precomputed optical thickness. Examples: $25 \times 25 \times 25$ for simple models, $60 \times 60 \times 60$ for complex models [S1, unofficial].
- **Map Resolution:** Resolution of the VNM, Irradiance Map, and LUT. Example: $1024 \times 1024$ [S1, unofficial].
- **Sampling Filter:** 21-taps filter for the radiance integral [S1, unofficial].

### Edge Cases and Pitfalls
- **Concave Shapes:** TSM fails here due to thickness calculation errors; this method handles them correctly by using precomputed optical thickness [S1, unofficial].
- **Internal Cavities:** Multi-view Depth Maps fail here; this method handles them correctly [S1, unofficial].
- **Heterogeneous Media:** The method supports different extinction coefficients for different media layers, which other methods like Multi-view Depth Maps do not [S1, unofficial].
- **Assumption Violation:** If in-scattering is significant or internal reflections/refractions are strong, the results will be inaccurate as these effects are neglected [S1, unofficial].

## Operating characteristics
- **Time Complexity:** Precomputation is $O(N^2)$ where $N$ is the number of surface voxels, due to pairwise optical thickness computation. Runtime is dominated by the 21-taps sampling filter and texture lookups, which is constant per pixel relative to scene complexity but depends on map resolution.
- **Memory Complexity:** The LUT size is $O(N^2)$ texels. For a $25 \times 25 \times 25$ voxel grid with a surface voxel count $N$, the LUT size can be significant. The VNM and Irradiance Map are $O(R^2)$ where $R$ is the map resolution [S1, unofficial].
- **Measured Performance:**
    - Translucent cube with spherical cavity (2,020 vertices): Voxel division $25 \times 25 \times 25$, FPS 405, map resolution $1024 \times 1024$ [S1, unofficial].
    - Translucent bottle with wine (10,604 vertices): Voxel division $60 \times 60 \times 60$, FPS 238, map resolution $1024 \times 1024$ [S1, unofficial].
    - Hardware: Intel Core i5-4200M 2.50 GHz, Intel HD Graphics 4600 [S1, unofficial].
- **Strengths:** Real-time performance for complex heterogeneous translucent objects; handles concave shapes and internal cavities accurately; supports multiple media types.
- **Limitations:** Assumes single scattering; high memory usage for the LUT for large surface voxel counts; precomputation cost; limited to static or slowly changing geometry (precomputation must be redone if geometry changes) [S1, unofficial].
- **When to use:** Real-time rendering of heterogeneous translucent objects with cavities (e.g., plastic bottles, jellyfish, flowers) where single scattering dominates.
- **When not to use:** Objects with significant in-scattering, strong internal reflections/refractions, or dynamic geometry requiring frequent precomputation updates [S1, unofficial].

## Background
The method was introduced by Keisuke Mochida, Midori Okamoto, Hiroyuki Kubo, and Shigeo Morishima in EUROGRAPHICS 2016 [S1, unofficial]. It was implemented in C++ and DirectX11 [S1, unofficial]. It addresses the limitations of Translucent Shadow Maps (TSM) [DS03] and Multi-view Depth Maps [KHKM12] in handling concave shapes and internal cavities [S1, unofficial].

## Sources
[S1] Real-time Rendering of Heterogeneous Translucent Objects using Voxel Number Map, https://doi.org/10.2312/egp20161037, fetched 2026-11-03 (unofficial)
