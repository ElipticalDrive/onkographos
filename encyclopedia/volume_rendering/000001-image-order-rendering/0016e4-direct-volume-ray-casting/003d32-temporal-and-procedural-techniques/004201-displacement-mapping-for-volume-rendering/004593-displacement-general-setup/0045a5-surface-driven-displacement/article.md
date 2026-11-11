# Surface Driven Displacement

Also called: Monochrome Texture Displacement

## Overview
Surface Driven Displacement is a technique for adding geometric detail to volume rendering by shifting fluid points based on a monochrome texture map. It solves the problem of inconsistent displacement directions in volumetric data by projecting points onto the fluid's isosurface and displacing them along the surface normal, ensuring that points above and below the surface move in the same direction. This distinguishes it from Gradient Driven Displacement, where displacement direction varies per point based on the local gradient, and Vector Displacement, which uses a colored vector texture.

## How it works
**Inputs and Outputs:**
*   **Input:** A volume dataset containing a defined Surface channel, a monochrome texture map, and the current position of each volume point.
*   **Output:** A displaced volume where points are shifted along the normal of their projection on the isosurface.

**Data Structures:**
*   **Surface Channel:** Defines the scalar field for the isosurface. The specific channel depends on the simulation type (see Subtopics).
*   **Monochrome Texture:** A 2D or 3D texture map where pixel/voxel brightness determines the displacement magnitude.

**Steps:**
1.  **Identify Surface Channel:** Determine the scalar field defining the fluid surface based on the `Surface Channel` parameter. Let this field be $S(x, y, z)$.
2.  **Define Isosurface Level:** Determine the isosurface threshold value, $L$. Points where $S(x, y, z) = L$ lie on the surface.
3.  **Project Point:** For a given volume point $P$, find its projection $P_{proj}$ onto the isosurface defined by $S(x, y, z) = L$. This is typically done by finding the nearest point on the isosurface or interpolating along the gradient direction from $P$ to the surface.
4.  **Compute Surface Normal:** Calculate the normal vector $\mathbf{n}$ at $P_{proj}$. The normal is the normalized gradient of the Surface channel at the projection point:
    $$ \mathbf{n} = \frac{\nabla S(P_{proj})}{\|\nabla S(P_{proj})\|} $$
    [G] If the gradient is zero (flat region), the normal is undefined; in this case, use a default normal (e.g., view direction or zero vector) or skip displacement.
5.  **Sample Texture:** Sample the monochrome texture at the location of the projection point $P_{proj}$. Let the sampled brightness value be $D$. The value $D$ is typically in the range $[0, 1]$.
6.  **Calculate Displacement:** Compute the displacement vector $\mathbf{d}$:
    $$ \mathbf{d} = D \cdot \mathbf{n} $$
7.  **Apply Displacement:** Shift the original point $P$ by $\mathbf{d}$:
    $$ P_{new} = P + \mathbf{d} $$

**Parameters:**
*   **Surface Channel:** Specifies which volume channel defines the surface (e.g., Liquid, Smoke, Speed, Fuel, Texture).
*   **Isosurface Level:** The threshold value for the surface. Typical ranges: 0-1 for Liquid/Smoke, 600-2000 for Fire/Smoke Temperature.
*   **Texture Map:** The monochrome texture providing displacement magnitude.
*   **Invert Volume:** A boolean flag. If enabled, swaps inside/outside, affecting which side of the surface is considered for displacement.

**Edge Cases and Pitfalls:**
*   **Precision:** Accurate projection and gradient calculation are critical. Poor resolution in the volume or texture can lead to jagged displacement.
*   **Boundaries:** Points far from the surface may have ambiguous projections. The method assumes points are reasonably close to the surface.
*   **Performance:** This method is slower than Gradient Driven Displacement because it requires finding the projection point and computing the gradient at that new location for every displaced point.

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def surface_driven_displacement(point P, volume_data, surface_channel, isosurface_level, texture_map):
    # 1. Get the scalar value at the point
    s_val = volume_data.get_value(P, surface_channel)
    
    # 2. Find projection P_proj onto isosurface
    # This involves finding the nearest point on the isosurface S(x,y,z) = isosurface_level
    # Implementation detail: Use a root-finding algorithm or nearest neighbor search
    P_proj = find_isosurface_projection(P, volume_data, surface_channel, isosurface_level)
    
    # 3. Compute gradient at projection point
    grad = volume_data.compute_gradient(P_proj, surface_channel)
    
    # Handle degenerate case where gradient is zero
    if magnitude(grad) < epsilon:
        return P # No displacement
    
    # 4. Normalize gradient to get normal
    normal = normalize(grad)
    
    # 5. Sample texture at projection point
    displacement_magnitude = texture_map.sample(P_proj)
    
    # 6. Calculate displacement vector
    displacement = displacement_magnitude * normal
    
    # 7. Return displaced point
    return P + displacement
```

## Subtopics

### Surface Channel Definition
The Surface Channel defines the scalar field used to determine the fluid's surface for solid rendering and displacement. The choice of channel depends on the simulation type and available data.
*   **Texture:** Uses a custom texture map to define the mesh surface. The texture values completely replace cache files in Mesh, Ocean Mesh, Cap Mesh, and Isosurface render modes.
*   **Liquid/Temperature:** Uses the Liquid or Temperature channel. For Liquid simulations, the temperature range is typically 0-1. For Fire/Smoke, the temperature range is typically 600-2000.
*   **Smoke:** Uses the Smoke channel. The smoke density range is typically 0-1 for Fire/Smoke simulations.
*   **Speed:** Uses the Speed channel, which is the length of the velocity vector. Requires the Speed channel output to be enabled in the simulation.
*   **Fuel:** Uses the Fuel channel. Requires the Fuel channel output to be enabled in the simulation.

## Operating characteristics
*   **Time Complexity:** Slower than Gradient Driven Displacement. Requires an additional step to project points to the isosurface and compute gradients at the projection point.
*   **Memory Complexity:** Requires storage for the volume data, the texture map, and potentially intermediate data for projection.
*   **Strengths:** Produces displacement results more similar to displaced solid geometry surfaces, as all points above or below the surface are displaced in the same direction (along the surface normal). Suitable for smoke and fire simulations where consistent surface deformation is desired.
*   **Limitations:** Computationally expensive. Performance degrades with complex geometries or high-resolution textures.
*   **When to Use:** When visual consistency with solid geometry displacement is required, especially for fire and smoke.
*   **When Not to Use:** When performance is critical and the visual difference from Gradient Driven Displacement is negligible.

## Background
Surface Driven Displacement is a technique used in Phoenix for 3ds Max for rendering fire and smoke simulations. It is part of the displacement mapping capabilities that allow adding detail to the simulation during rendering.

## Sources
[S1] FireSmoke Rendering - Phoenix for 3ds Max, https://documentation.chaos.com/space/PHX4MAX/124396558, fetched 2026-10-27
