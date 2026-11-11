# Displacement General Setup

Also called: Discontinuous Displacement Mapping, Generalized Displacement Mapping, Inverse Displacement Mapping

## Overview
Displacement General Setup is a rendering-time technique for adding geometric detail to volumetric simulations by shifting fluid points or surface vertices based on sampled texture maps [S1]. It solves the limitation of static volumetric meshes by allowing complex, high-frequency surface deformations (such as waves or smoke turbulence) without re-tessellating the underlying geometry [S1]. It distinguishes itself from standard mesh displacement by operating on volumetric data structures (like Phoenix fluid grids) and supporting specialized modes like Surface-driven and Vector displacement, which handle directional consistency differently than scalar or gradient-based methods [S1].

## How it works
The method applies a displacement map to a volumetric object during the rendering phase. The core logic involves sampling a texture at the location of a fluid point or surface vertex and shifting that point along a specific direction vector by a distance determined by the texture's value.

### Inputs and Outputs
*   **Inputs**:
    *   Volumetric data (fluid surface or volume grid).
    *   Displacement map (texture). Can be monochrome (grayscale) or colored (RGB) [S1].
    *   Parameters: `displacement` (boolean enable), `displmul` (multiplier), `Type` (algorithm selection), `Map` (texture reference) [S1].
*   **Outputs**:
    *   Rendered image with displaced geometry.
    *   Mesh preview (if enabled) showing the deformed surface [S1].

### Data Structures
The implementation relies on the following parameters, typically stored in the renderer's material or simulation context:
*   `displacement`: Boolean flag. If false, displacement is skipped.
*   `displmul`: Float multiplier. Scales the final displacement magnitude.
*   `Type`: Enum or string identifying the displacement algorithm. Common values: `Gradient driven`, `Surface driven`, `Vector` [S1].
*   `Map`: Reference to the texture object. The renderer must support V-Ray, 3ds Max, or Phoenix texture formats [S1].

### Steps
1.  **Enable Check**: If `displacement` is false, skip displacement processing.
2.  **Mode Detection**: Determine the rendering mode.
    *   If Mode is `Mesh`, `Ocean Mesh`, or `Cap Mesh`, the `Type` parameter is ignored [S1].
    *   In these modes, Phoenix automatically detects the map type:
        *   If `Map` is monochrome, use **Surface-driven** displacement [S1].
        *   If `Map` is colored, use **Vector** displacement [S1].
    *   If Mode is not one of the above, use the explicit `Type` parameter to select the algorithm [S1].
3.  **Texture Sampling**: For each point $P$ on the fluid surface or volume:
    *   Sample the displacement map at $P$'s texture coordinates.
    *   Let $T$ be the sampled value.
    *   If the map is monochrome, $T$ is a scalar brightness value.
    *   If the map is colored, $T$ is a 3D vector $(R, G, B)$ [S1].
4.  **Direction Calculation**: Determine the displacement direction vector $\vec{D}$ based on the active algorithm:
    *   **Gradient driven**: Calculate the gradient of the fluid's Surface channel at $P$. Let $\nabla S$ be this gradient. $\vec{D} = \frac{\nabla S}{\|\nabla S\|}$ [S1].
    *   **Surface driven**: Project $P$ onto the isosurface of the fluid's Surface channel. Let $P_{proj}$ be the projection. Calculate the surface normal $\vec{N}$ at $P_{proj}$. $\vec{D} = \vec{N}$ [S1].
    *   **Vector**: Use the sampled color $T$ directly as the direction vector. $\vec{D} = T$ [S1].
5.  **Magnitude Calculation**:
    *   If the map is monochrome, the base magnitude is $T$ [S1].
    *   If the map is colored (and used as a vector map), the magnitude is determined by the total intensity of the color. Let $I = R + G + B$ (or a weighted sum, depending on implementation details not specified in sources). The magnitude is $I$ [S1].
    *   Apply the multiplier: $M = \text{displmul} \times \text{base\_magnitude}$ [S1].
6.  **Displacement Application**:
    *   Calculate the final displacement vector: $\vec{V} = M \times \vec{D}$ [S1].
    *   Shift the point: $P_{new} = P + \vec{V}$ [S1].
7.  **Edge Case Handling**:
    *   If a colored map is specified when a monochrome map is needed (e.g., Gradient or Surface driven mode), the strength is determined by the total intensity of the color [S1].
    *   If a monochrome map is specified when a vector map is needed, the entire displacement points in a single direction (the default vector, likely $(1,0,0)$ or similar, though the exact default vector is not specified in the sources) [S1].

### Pseudocode
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def apply_displacement(point_p, texture_map, mode, type_param, displmul, displacement_enabled):
    if not displacement_enabled:
        return point_p

    # Step 2: Mode Detection
    if mode in ['Mesh', 'Ocean Mesh', 'Cap Mesh']:
        # Type is ignored; auto-detect based on map type
        is_colored = texture_map.is_colored()
        if is_colored:
            active_type = 'Vector'
        else:
            active_type = 'Surface driven'
    else:
        active_type = type_param

    # Step 3: Texture Sampling
    sampled_value = texture_map.sample(point_p)

    # Step 4 & 5: Direction and Magnitude Calculation
    if active_type == 'Gradient driven':
        # Requires monochrome map
        # Magnitude is brightness
        magnitude = sampled_value.brightness
        # Direction is gradient of Surface channel
        gradient = fluid_surface_gradient(point_p)
        direction = normalize(gradient)
    elif active_type == 'Surface driven':
        # Requires monochrome map
        # Magnitude is brightness
        magnitude = sampled_value.brightness
        # Project point to isosurface
        point_proj = project_to_isosurface(point_p, fluid_surface)
        # Direction is normal at projection
        direction = surface_normal(point_proj, fluid_surface)
    elif active_type == 'Vector':
        # Requires colored map
        # Direction is the color vector
        direction = sampled_value.rgb
        # Magnitude is total intensity of color
        magnitude = sampled_value.r + sampled_value.g + sampled_value.b
    else:
        raise ValueError(f"Unknown displacement type: {active_type}")

    # Apply multiplier
    final_magnitude = magnitude * displmul

    # Step 6: Displacement Application
    displacement_vector = final_magnitude * direction
    new_point = point_p + displacement_vector

    return new_point
```

### Parameters
*   `displacement`: Boolean. Enables the effect. Default: False [S1].
*   `displmul`: Float. Multiplier for displacement amount. Default: 1.0 (implied, not explicitly stated) [S1].
*   `Type`: Enum. `Gradient driven`, `Surface driven`, `Vector`. Default: Not specified [S1].
*   `Map`: Texture reference. Can be V-Ray, 3ds Max, or Phoenix format. Default: None [S1].

### Pitfalls
*   **Mode/Type Conflict**: The `Type` parameter is ignored in `Mesh`, `Ocean Mesh`, and `Cap Mesh` modes. Users must ensure the map type matches the expected behavior (monochrome for Surface-driven, colored for Vector) [S1].
*   **Map Mismatch**: Using a colored map in a monochrome mode results in displacement strength based on color intensity, which may be unintended. Using a monochrome map in a vector mode results in uniform direction displacement [S1].
*   **Precision**: Not specified in the sources.

## Subtopics

### [Vector Displacement](11532-vector-displacement.md)
Uses a colored vector texture to shift points, enabling asymmetric geometries like waves with convex backs and concave fronts. Choose when directional control of displacement magnitude and orientation is required.

### [Surface Driven Displacement](11547-surface-driven-displacement.md)
Uses a monochrome texture to shift points along the surface normal of the projected isosurface. Choose when consistent displacement direction relative to the surface is needed, avoiding the varying directions of gradient-driven methods.

## Operating characteristics
*   **Time Complexity**: Not given in the sources. Depends on texture sampling cost and surface projection cost (for Surface-driven).
*   **Memory Complexity**: Not given in the sources. Depends on texture map size and fluid grid resolution.
*   **Measured Performance**: Not given in the sources.
*   **Strengths**: Adds high-frequency detail without modifying the simulation cache; supports complex geometries via Vector displacement; integrates with existing texture workflows [S1].
*   **Limitations**: Rendering-time only (does not affect simulation physics); potential for visual artifacts if map mismatch occurs; performance impact depends on texture resolution and sampling cost [S1].
*   **When to Use**: When visual detail is needed for smoke, fire, or liquid surfaces without re-simulating or re-meshing [S1].
*   **When Not to Use**: When the displacement needs to affect the simulation physics or collision detection [S1].

## Background
Displacement mapping in volumetric rendering, specifically within the Phoenix FD context, was introduced as a method to enhance visual fidelity by shifting fluid points during rendering [S1]. The technique draws inspiration from standard geometry displacement but adapts it for volumetric data structures [S1]. It is used in Phoenix for 3ds Max to allow artists to add detail to explosions, smoke, and fire simulations interactively [S1].

## Sources
[S1] FireSmoke Rendering - Phoenix for 3ds Max, https://documentation.chaos.com/space/PHX4MAX/124396558, fetched 2026-10-27
