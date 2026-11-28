# Surface Driven Displacement

Also called: Monochrome Texture Displacement

## Overview
Surface Driven Displacement is a displacement technique that shifts fluid points along the normal of a specific isosurface rather than the local density gradient [S1]. It ensures that all points above or below the fluid surface are displaced in the same direction, producing results more similar to displaced solid geometry surfaces [S1]. It is distinct from Gradient-driven displacement, which uses the local density gradient for potentially varying directions, and Vector displacement, which uses a colored vector texture map [S1].

## How it works
**Inputs:**
1. Volumetric simulation data containing a defined "Surface" channel [S1].
2. A monochrome texture map [S1].

**Outputs:**
1. Displaced volumetric geometry [S1].

**Data Structures:**
* Surface Channel: Scalar field defining the fluid boundary [S1]. Exact layouts, sizes, and bit packing: Not given in the sources [S1].
* Monochrome Texture: Texture where displacement magnitude is encoded in brightness at each point [S1]. Exact layouts, sizes, and bit packing: Not given in the sources [S1].

**Steps:**
1. For a given point $P$ in the fluid volume, determine its projection $P_{\text{proj}}$ onto the isosurface of the defined Surface channel [S1].
2. Calculate the surface normal $\mathbf{N}$ at $P_{\text{proj}}$ [S1].
3. Sample the monochrome texture at $P_{\text{proj}}$ to obtain the displacement magnitude $D$, which corresponds to the texture's brightness at that location [S1].
4. Shift the original point $P$ along the normal $\mathbf{N}$ by the distance $D$ to compute the new position $P'$:
   $$ P' = P + D \cdot \mathbf{N} $$
   The sources specify shifting along the normal of the projection point rather than the gradient at the original point $P$ [S1].

**Parameters:**
* Surface channel selection and monochrome texture mapping are required [S1].
* Typical values: Not given in the sources [S1].

**Edge Cases and Pitfalls:**
* **Performance:** This method is slower than Gradient-driven displacement because it requires finding the projection onto the isosurface and calculating the normal at that projection point [S1].
* **Direction Consistency:** Unlike gradient-driven methods, this method ensures that points above and below the surface are displaced in the same direction relative to the surface normal, which is critical for smoke and fire simulations to maintain visual coherence [S1].

## Subtopics

### [Surface Channel Definition](12817-surface-channel-definition.md)
Specifies which data channel defines the fluid's surface geometry and the isosurface threshold. It relies on a specific channel's scalar values to establish the surface reference rather than computing normals from density gradients.

## Operating characteristics
* **Time Complexity:** Slower than Gradient-driven displacement due to the additional computation required to find the surface projection and normal [S1].
* **Memory Complexity:** Requires storage for the Surface channel data and the monochrome texture map [S1].
* **Strengths:** Produces displacement results more similar to displaced solid geometry surfaces [S1]. It ensures consistent displacement direction for points above and below the surface, making it suitable for smoke and fire simulations [S1].
* **Limitations:** Computationally more expensive than Gradient-driven displacement [S1].
* **When to use:** When simulating smoke or fire where consistent surface-normal-based displacement is desired for visual realism [S1].
* **When not to use:** When performance is critical and the directional variation of Gradient-driven displacement is acceptable or preferred [S1].

## Background
Surface Driven Displacement is a feature within Phoenix for 3ds Max, part of the V-Ray ecosystem [S1]. It is designed to handle the specific requirements of fluid simulations like smoke and fire, where standard gradient-based displacement might produce visually inconsistent results due to varying local gradients [S1]. The method relies on the concept of a "Surface Channel" to define a clear boundary for displacement operations [S1].

## Sources
[S1] FireSmoke Rendering - Phoenix for 3ds Max, https://documentation.chaos.com/space/PHX4MAX/124396558, fetched 2026-10-27
