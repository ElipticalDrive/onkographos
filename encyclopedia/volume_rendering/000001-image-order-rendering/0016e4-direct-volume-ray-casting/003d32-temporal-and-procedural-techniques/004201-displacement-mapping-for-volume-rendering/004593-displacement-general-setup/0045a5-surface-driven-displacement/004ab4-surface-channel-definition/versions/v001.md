# Surface Channel Definition

Also called: Surface Channel

## Overview
The Surface Channel Definition specifies which data channel within a volumetric simulation defines the fluid's surface geometry for solid rendering and displacement mapping [S1]. It solves the problem of determining the boundary between the fluid and the environment by selecting a scalar field (such as Liquid, Smoke, or Speed) and an isosurface threshold [S1]. It is distinct from Gradient-driven displacement because it relies on a specific channel's scalar value rather than the gradient of the density field to determine surface normals and displacement directions [S1].

## How it works
The method selects a source channel to act as the implicit surface definition. The renderer or geometry generator evaluates this channel at each voxel or sample point. If the value exceeds a defined threshold (the isosurface level), the point is considered part of the surface or interior, depending on the `Invert volume` setting [S1].

**Inputs and Outputs:**
*   **Input:** A volumetric dataset containing multiple channels (e.g., Liquid, Smoke, Temperature, Speed, Fuel, or a custom Texture map).
*   **Input:** A scalar threshold value (`surflevel`) [S1].
*   **Input:** A boolean flag for inversion (`solidbelow`) [S1].
*   **Output:** A binary or scalar field representing the surface boundary, used for mesh generation (Mesh, Ocean Mesh, Cap Mesh, Isosurface modes) or as the reference surface for displacement [S1].

**Data Structures:**
The specific internal data structure layout is not given in the sources. The method relies on the existence of named channels within the simulation cache.

**Parameters:**
*   **Surface Channel (`sarg`):** The identifier of the channel defining the surface. Options include:
    *   `Liquid/Temperature`: Defines the liquid surface. Typical ranges are 0–1 for Liquid simulations and 600–2000 for Fire/Smoke simulations [S1].
    *   `Smoke`: Defines the liquid surface. Typical range is 0–1 for Fire/Smoke simulations [S1].
    *   `Speed`: Defines the liquid surface. Calculated as the length of the velocity vector for each voxel. Requires the Speed channel output to be enabled in the simulation settings [S1].
    *   `Fuel`: Defines the liquid surface. Requires the Fuel channel output to be enabled in the simulation settings [S1].
    *   `Texture`: Uses a custom texture map to define the mesh surface. Requires the `stex` slot to specify the texture file [S1].
*   **Isosurface Level (`surflevel`):** The threshold value for geometry generation. Grid cells with values below this threshold are ignored (or treated as outside, depending on inversion) [S1].
*   **Invert Volume (`solidbelow`):** If enabled, swaps the inside and outside definitions. By default, values above the surface level are considered internal [S1].

**Steps:**
1.  Identify the selected `Surface Channel` (`sarg`) from the available simulation outputs.
2.  If `sarg` is `Texture`, load the texture specified in the `stex` slot. Note that in Mesh, Ocean Mesh, Cap Mesh, and Isosurface render modes, this map completely replaces loaded cache files [S1].
3.  For non-texture channels, ensure the corresponding channel output is enabled in the simulation (required for `Speed` and `Fuel`) [S1].
4.  Evaluate the scalar value $V$ at each voxel position $\mathbf{p}$ from the selected channel.
5.  Compare $V$ against the `Isosurface Level` $T$.
6.  Determine surface membership:
    *   If `Invert volume` is disabled: Points where $V \ge T$ are internal/surface; points where $V < T$ are external.
    *   If `Invert volume` is enabled: Points where $V < T$ are internal/surface; points where $V \ge T$ are external.
7.  Generate the surface geometry or provide the surface field for displacement calculations [S1].

**Edge Cases and Pitfalls:**
*   **Missing Outputs:** The `Speed` and `Fuel` channels will not function if their respective outputs are not explicitly enabled in the simulation settings [S1].
*   **Texture Replacement:** When using the `Texture` mode in Mesh, Ocean Mesh, Cap Mesh, or Isosurface modes, the selected texture map completely replaces any loaded cache files, which may discard simulation data [S1].
*   **Range Mismatch:** Using default thresholds with channels that have non-standard ranges (e.g., using a 0–1 threshold for a Temperature channel in Fire/Smoke simulations which ranges 600–2000) will result in incorrect surface generation [S1].

## Operating characteristics
Not given in the sources.

## Background
The concept is based on the isosurface generation technique, which is unique for generating surfaces from volumetric data [S1]. While primarily used for rendering liquids, it is also applied to smoke and fire to create effects such as underwater bubbles or freezing fire [S1]. This method is part of the Phoenix for 3ds Max rendering pipeline [S1].

## Sources
[S1] FireSmoke Rendering - Phoenix for 3ds Max, https://documentation.chaos.com/space/PHX4MAX/124396558, fetched 2026-10-27
