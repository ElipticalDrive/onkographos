# Surface Driven Displacement

Also called: Monochrome Texture Displacement

## Overview
Surface Driven Displacement is a technique for adding geometric detail to volumetric simulations by shifting fluid points along the normals of a specific isosurface, rather than the local density gradient. It solves the problem of directional inconsistency in displacement by ensuring that all points (both inside and outside the fluid) are displaced in the same global direction relative to the surface, producing results that resemble displaced solid geometry. It is distinct from Gradient-driven displacement, which uses the local density gradient for potentially varying directions, and Vector displacement, which uses a colored vector texture.

## How it works
**Inputs:**
1.  Volumetric simulation data containing a defined "Surface" channel (scalar field) [S1].
2.  A monochrome texture map [S1].

**Outputs:**
1.  Displaced volumetric geometry suitable for rendering [S1].

**Data Structures:**
*   **Surface Channel:** A scalar field defining the fluid's boundary. The specific channel (e.g., Liquid, Smoke, Speed) and its isosurface threshold must be defined beforehand [S1].
*   **Monochrome Texture:** A 2D or 3D texture where the displacement magnitude is encoded in the brightness (intensity) of the texture at a given coordinate [S1].

**Steps:**
1.  **Identify Surface Normal:** For a given point $P$ in the fluid volume, determine its projection $P_{proj}$ onto the isosurface of the defined Surface channel. Calculate the surface normal $\mathbf{N}$ at $P_{proj}$ [S1].
2.  **Sample Displacement Magnitude:** Sample the monochrome texture at the projected point $P_{proj}$ to obtain the displacement magnitude $D$. The value $D$ corresponds to the brightness of the texture at that location [S1].
3.  **Apply Displacement:** Shift the original point $P$ along the normal $\mathbf{N}$ by the distance $D$. The new position $P'$ is calculated as:
    $$ P' = P + D \cdot \mathbf{N} $$
    Note: The sources specify shifting along the normal of the projection point, not the gradient at the original point $P$ [S1].

**Parameters:**
*   **Surface Channel Selection:** The user must select which scalar field defines the surface (e.g., Liquid, Smoke) [S1].
*   **Isosurface Threshold:** The specific value within the selected channel that defines the surface boundary [S1].
*   **Texture Mapping:** The monochrome texture must be mapped appropriately to the simulation domain [S1].

**Edge Cases and Pitfalls:**
*   **Performance:** This method is slower than Gradient-driven displacement because it requires finding the projection onto the isosurface and calculating the normal at that projection point, rather than simply evaluating the gradient at the current point [S1].
*   **Direction Consistency:** Unlike gradient-driven methods, this method ensures that points above and below the surface are displaced in the same direction relative to the surface normal, which is critical for smoke and fire simulations to maintain visual coherence [S1].

## Subtopics

### [Surface Channel Definition](12817-surface-channel-definition.md)
Specifies which data channel defines the fluid's surface geometry and the isosurface threshold. It is distinct from Gradient-driven displacement because it relies on a specific channel's scalar value rather than the gradient of the density field.

## Operating characteristics
*   **Time Complexity:** Slower than Gradient-driven displacement due to the additional computation required to find the surface projection and normal [S1].
*   **Memory Complexity:** Requires storage for the Surface channel data and the monochrome texture map [S1].
*   **Strengths:** Produces displacement results more similar to displaced solid geometry surfaces; ensures consistent displacement direction for points above and below the surface, making it suitable for smoke and fire simulations [S1].
*   **Limitations:** Computationally more expensive than Gradient-driven displacement [S1].
*   **When to use:** When simulating smoke or fire where consistent surface-normal-based displacement is desired for visual realism [S1].
*   **When not to use:** When performance is critical and the directional variation of Gradient-driven displacement is acceptable or preferred [S1].

## Background
Surface Driven Displacement is a feature within Phoenix for 3ds Max, part of the V-Ray ecosystem. It is designed to handle the specific requirements of fluid simulations like smoke and fire, where standard gradient-based displacement might produce visually inconsistent results due to varying local gradients. The method relies on the concept of a "Surface Channel" to define a clear boundary for displacement operations.

## Sources
[S1] FireSmoke Rendering - Phoenix for 3ds Max, https://documentation.chaos.com/space/PHX4MAX/124396558, fetched 2026-10-27
