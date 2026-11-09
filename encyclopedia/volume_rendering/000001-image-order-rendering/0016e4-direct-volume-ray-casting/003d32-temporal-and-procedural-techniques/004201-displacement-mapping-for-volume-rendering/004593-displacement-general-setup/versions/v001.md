# Displacement General Setup

Also called: Discontinuous Displacement Mapping, Generalized Displacement Mapping, Inverse Displacement Mapping

## Overview
Displacement General Setup is a rendering technique that adds geometric detail to volumetric simulations by sampling a texture map and shifting fluid or surface points in a direction determined by that texture [S1]. It solves the limitation of static volumetric representations by allowing interactive deformation without mesh re-tessellation, building on the principles of [Displacement Mapping for Volume Rendering](10659-displacement-mapping-for-volume-rendering.md). Unlike standard scalar displacement which often produces symmetrical deformations, this setup supports vector-based and surface-normal-based shifts, enabling complex, asymmetric geometries such as waves with convex backs and concave fronts [S1].

## How it works
**Inputs and Outputs:**
*   **Input:** A volumetric simulation (fluid, smoke, or fire) defined by a density field or isosurface. A displacement texture map (monochrome or colored) [S1].
*   **Output:** A rendered volume or mesh with displaced geometry, where point positions are altered based on the texture values [S1].

**Data Structures:**
The displacement map is a texture resource compatible with V-Ray, 3ds Max, or Phoenix standards [S1]. It can be:
1.  **Monochrome:** Single channel (grayscale), representing scalar magnitude [S1].
2.  **Colored (Vector):** Three channels (RGB), representing a 3D displacement vector [S1].

**Parameters:**
*   `displacement` (Boolean): Enables the displacement effect [S1].
*   `displmul` (Float): Multiplier value for the displacement amount [S1].
*   `Type` (Enum): Specifies the displacement technique (Gradient driven, Surface driven, or Vector) [S1].
*   `Map` (`displ2`): The texture map resource [S1].

**Algorithm:**
The displacement process follows these steps:

1.  **Enable Check:** If `displacement` is false, render the volume without modification [S1].
2.  **Mode Detection:** Determine the rendering mode. If the mode is `Mesh`, `Ocean Mesh`, or `Cap Mesh`, the `Type` parameter is ignored. Phoenix automatically detects the map type: monochrome maps trigger Surface-driven displacement, and colored maps trigger Vector displacement [S1].
3.  **Texture Sampling:** Sample the displacement map at the current point's spatial coordinates to obtain a value $V$ [S1].
    *   If the map is monochrome, $V$ is a scalar brightness value.
    *   If the map is colored, $V$ is a vector $\vec{D} = (R, G, B)$ [S1].
4.  **Direction Calculation:** Determine the displacement direction $\vec{d}$ based on the selected `Type` or auto-detected mode:
    *   **Gradient Driven:** Calculate the gradient of the fluid's Surface channel at the current point. The direction $\vec{d}$ is the normalized gradient vector $\nabla S$ [S1].
    *   **Surface Driven:** Project the current point onto the isosurface of the fluid's Surface channel. Calculate the normal $\vec{n}$ at this projection point. The direction $\vec{d}$ is $\vec{n}$ [S1].
    *   **Vector:** The direction $\vec{d}$ is derived from the vector texture $V$ itself [S1].
5.  **Magnitude Calculation:**
    *   If using a monochrome map, the magnitude is the texture brightness $V$ scaled by `displmul` [S1].
    *   If using a colored map for a monochrome-required mode, the magnitude is the total intensity of the color (e.g., sum or norm of RGB channels) [S1].
    *   If using a colored map for Vector mode, the magnitude is implicitly handled by the vector components [S1].
6.  **Point Shift:** Shift the point $P$ to $P'$ by adding the displacement vector:
    $$ P' = P + \text{displmul} \cdot V \cdot \vec{d} $$
    *Note: For Vector displacement, $V$ is the vector $\vec{D}$, so the formula simplifies to $P' = P + \text{displmul} \cdot \vec{D}$ [S1].*
7.  **Projection Sampling (Surface Driven Only):** For Surface-driven displacement, the texture is also sampled at the projection point on the isosurface to determine the final magnitude, ensuring consistency across the surface thickness [S1].

**Edge Cases and Pitfalls:**
*   **Mode Ignorance:** The `Type` parameter is ignored in `Mesh`, `Ocean Mesh`, and `Cap Mesh` modes [S1].
*   **Map Mismatch:**
    *   If a colored map is specified when a monochrome map is needed, the displacement strength is determined by the total intensity of the color [S1].
    *   If a monochrome map is specified when a vector map is needed, the entire displacement points in a single direction (likely the gradient or normal, scaled by the scalar value) [S1].
*   **Performance:** Surface-driven displacement is slower than Gradient-driven displacement [S1].

## Subtopics

### Surface Driven Displacement
A displacement technique requiring a monochrome texture map. The displacement amount is the texture's brightness at each point. Each point of the fluid is shifted along the normal of the point's projection on the isosurface of the fluid's Surface channel. The texture is also sampled at the projection point. This ensures all points above or below the fluid surface are displaced in the same direction, producing results more similar to displaced solid geometry surfaces. It is slower than Gradient driven displacement [S1].

### Gradient Driven Displacement
Requires a monochrome texture map. The displacement amount is the texture's brightness at each point. Each point of the fluid is shifted along the gradient of the Surface channel. This means that each point in space could have a different displacement direction. This method is suitable for smoke and fire [S1].

### [Vector Displacement](11532-vector-displacement.md)
Vector Displacement is a procedural technique in volume rendering that shifts fluid or surface points based on a colored vector texture map, where the texture's RGB channels are interpreted as a 3D displacement vector. It solves the limitation of scalar displacement methods by enabling the creation of asymmetric, complex surface geometries, such as waves with convex backs and concave fronts, which scalar methods cannot produce. Unlike Surface-driven displacement, which relies on gradient fields and produces symmetrical forms, Vector Displacement allows for directional control of the displacement magnitude and orientation directly through the texture data.

## Operating characteristics
*   **Performance:** Surface-driven displacement is slower than Gradient-driven displacement [S1]. Vector displacement performance is not explicitly quantified but involves vector arithmetic.
*   **Strengths:** Enables complex, asymmetric geometries (e.g., waves with convex/concave sides) [S1]. Allows use of standard V-Ray, 3ds Max, or Phoenix texture maps [S1].
*   **Limitations:** Requires careful selection of map type (monochrome vs. colored) to avoid unintended single-direction displacement or intensity-based magnitude errors [S1]. Surface-driven mode is computationally expensive due to projection calculations [S1].
*   **When to use:** Use Vector displacement for asymmetric details like waves. Use Surface-driven for solid-like deformations on smoke/fire. Use Gradient-driven for general smoke/fire where directionality varies per point [S1].

## Background
Displacement in Phoenix for 3ds Max is a technique intended to add detail to the simulation during rendering, similar to usual geometry displacement [S1]. It allows users to plug in any V-Ray, 3ds Max, or Phoenix texture maps to shift the corresponding point of the fluid volume or surface [S1]. The feature includes options for Mesh Preview to check displacement effects on surfaces [S1].

## Sources
[S1] FireSmoke Rendering - Phoenix for 3ds Max, https://documentation.chaos.com/space/PHX4MAX/124396558, fetched 2026-10-27
