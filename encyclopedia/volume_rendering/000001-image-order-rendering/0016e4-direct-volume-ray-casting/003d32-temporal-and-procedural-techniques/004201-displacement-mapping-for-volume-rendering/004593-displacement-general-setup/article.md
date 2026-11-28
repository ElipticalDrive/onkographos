# Displacement General Setup

Also called: Discontinuous Displacement Mapping, Generalized Displacement Mapping, Inverse Displacement Mapping

## Overview
Displacement General Setup is a rendering-time technique for adding geometric detail to volumetric simulations by shifting fluid points or surface vertices based on sampled texture maps [S1]. It solves the limitation of static volumetric meshes by allowing complex, high-frequency surface deformations without re-tessellating the underlying geometry [S1]. It distinguishes itself from standard mesh displacement by operating directly on volumetric data structures and supporting specialized directional modes like Surface-driven and Vector displacement [S1].

## How it works
The method applies a displacement map to a volumetric object during the rendering phase [S1]. The core logic involves sampling a texture at the location of a fluid point or surface vertex and shifting that point along a specific direction vector by a distance determined by the texture's value [S1].

### Inputs and Outputs
*   **Inputs**:
    *   Volumetric data representing the fluid surface or volume grid [S1].
    *   Displacement map (texture), which can be monochrome (grayscale) or colored (RGB) [S1].
    *   Parameters: `displacement` (boolean enable), `displmul` (multiplier), `Type` (algorithm selection), and `Map` (texture reference) [S1].
*   **Outputs**:
    *   Rendered image with displaced geometry [S1].
    *   Mesh preview showing the deformed surface when the Mesh Preview option is enabled [S1].

### Data Structures
The implementation relies on the following parameters, typically stored in the renderer's material or simulation context:
*   `displacement`: Boolean flag. If false, displacement is skipped [S1].
*   `displmul`: Float multiplier. Scales the final displacement magnitude [S1].
*   `Type`: Enum or string identifying the displacement algorithm. Common values are `Gradient driven`, `Surface driven`, and `Vector` [S1].
*   `Map`: Reference to the texture object. The renderer must support V-Ray, 3ds Max, or Phoenix texture formats [S1].

### Steps
1.  **Enable Check**: If `displacement` is false, skip displacement processing [S1].
2.  **Mode Detection**: Determine the rendering mode [S1].
    *   If Mode is `Mesh`, `Ocean Mesh`, or `Cap Mesh`, the `Type` parameter is ignored [S1].
    *   In these modes, Phoenix automatically detects the map type: monochrome maps trigger **Surface-driven** displacement, and colored maps trigger **Vector** displacement [S1].
    *   If Mode is not one of the above, use the explicit `Type` parameter to select the algorithm [S1].
3.  **Texture Sampling**: For each point $P$ on the fluid surface or volume, sample the displacement map at $P$'s texture coordinates to obtain $T$ [S1].
    *   If the map is monochrome, $T$ is a scalar brightness value [S1].
    *   If the map is colored, $T$ is a 3D vector $(R, G, B)$ [S1].
4.  **Direction Calculation**: Determine the displacement direction vector $\vec{D}$ based on the active algorithm [S1].
    *   **Gradient driven**: Calculate the gradient of the fluid's Surface channel at $P$, denoted $\nabla S$. $\vec{D} = \frac{\nabla S}{\|\nabla S\|}$ [S1].
    *   **Surface driven**: Project $P$ onto the isosurface of the fluid's Surface channel to obtain $P_{proj}$. Calculate the surface normal $\vec{N}$ at $P_{proj}$. $\vec{D} = \vec{N}$ [S1].
    *   **Vector**: Use the sampled color vector from $T$ as the direction [S1].
5.  **Magnitude Calculation**:
    *   If the map is monochrome, the base magnitude is $T$ [S1].
    *   If the map is colored, the magnitude is determined by the total intensity of the color. Let $I = R + G + B$. The magnitude is $I$ [S1].
    *   Apply the multiplier: $M = \text{displmul} \times \text{base\_magnitude}$ [S1].
6.  **Displacement Application**: Calculate the final displacement vector $\vec{V} = M \times \vec{D}$ and shift the point: $P_{new} = P + \vec{V}$ [S1].
7.  **Edge Case Handling**:
    *   If a colored map is specified when a monochrome map is needed, the strength of the displacement is determined by the total intensity of the color [S1].
    *   If a monochrome map is specified when a vector map is needed, the entire displacement points in a single direction [S1].

### Pseudocode
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def apply_displacement(point_p, texture_map, mode, type_param, displmul, displacement_enabled):
    # Step 1: Enable Check
    if not displacement_enabled:
        return point_p

    # Step 2: Mode Detection
    if mode in ['Mesh', 'Ocean Mesh', 'Cap Mesh']:
        # Type is ignored; auto-detect based on map type
        is_colored = texture_map.is_colored()
        active_type = 'Vector' if is_colored else 'Surface driven'
    else:
        active_type = type_param

    # Step 3: Texture Sampling
    sampled_value = texture_map.sample(point_p)
    if active_type == 'Vector':
        T_vec = sampled_value.rgb
        base_magnitude = T_vec.r + T_vec.g + T_vec.b
    else:
        base_magnitude = sampled_value.brightness

    # Step 4: Direction Calculation
    if active_type == 'Gradient driven':
        # Step 4a: Gradient driven direction
        # Standard operation: compute gradient of fluid Surface channel at P [G]
        gradient = compute_gradient_of_surface_channel(point_p)
        direction = normalize(gradient)
    elif active_type == 'Surface driven':
        # Step 4b: Surface driven direction
        # Standard operation: project P onto isosurface of Surface channel [G]
        point_proj = project_to_isosurface(point_p)
        # Standard operation: compute surface normal at projection [G]
        direction = compute_surface_normal(point_proj)
    elif active_type == 'Vector':
        # Step 4c: Vector direction
        direction = normalize(T_vec)
    else:
        raise ValueError(f"Unknown displacement type: {active_type}")

    # Step 5: Magnitude Calculation
    final_magnitude = base_magnitude * displmul

    # Step 6: Displacement Application
    displacement_vector = final_magnitude * direction
    new_point = point_p + displacement_vector

    return new_point
```

### Parameters
*   `displacement`: Boolean. Enables the effect [S1].
*   `displmul`: Float. Multiplier for displacement amount [S1].
*   `Type`: Enum. `Gradient driven`, `Surface driven`, `Vector` [S1].
*   `Map`: Texture reference. Supports V-Ray, 3ds Max, or Phoenix formats [S1].

### Pitfalls
*   **Mode/Type Conflict**: The `Type` parameter is ignored in `Mesh`, `Ocean Mesh`, and `Cap Mesh` modes [S1].
*   **Map Mismatch**: Using a colored map in a monochrome mode results in displacement strength based on color intensity, which may be unintended [S1].
*   **Map Mismatch**: Using a monochrome map in a vector mode results in uniform direction displacement [S1].

## Subtopics

### [Vector Displacement](11532-vector-displacement.md)
Uses a colored vector texture to shift points, enabling asymmetric geometries like waves with convex backs and concave fronts. Choose when directional control of displacement magnitude and orientation is required.

### [Surface Driven Displacement](11547-surface-driven-displacement.md)
Uses a monochrome texture to shift points along the surface normal of the projected isosurface. Choose when consistent displacement direction relative to the surface is needed, avoiding the varying directions of gradient-driven methods.

## Operating characteristics
*   **Time Complexity**: Not given in the sources.
*   **Memory Complexity**: Not given in the sources.
*   **Measured Performance**: Not given in the sources.
*   **Strengths**: Adds high-frequency detail without modifying the simulation cache; supports complex geometries via Vector displacement; integrates with existing texture workflows [S1].
*   **Limitations**: Rendering-time only (does not affect simulation physics); potential for visual artifacts if map mismatch occurs; performance impact depends on texture resolution and sampling cost [S1].
*   **When to Use**: When visual detail is needed for smoke, fire, or fluid surfaces without re-simulating or re-meshing [S1].
*   **When Not to Use**: When the displacement needs to affect the simulation physics or collision detection [S1].

## Background
Displacement mapping in volumetric rendering, specifically within the Phoenix FD context, was introduced as a method to enhance visual fidelity by shifting fluid points during rendering [S1]. The technique draws inspiration from standard geometry displacement but adapts it for volumetric data structures [S1]. It is used in Phoenix for 3ds Max to allow artists to add detail to explosions, smoke, and fire simulations interactively [S1].

## Sources
[S1] FireSmoke Rendering - Phoenix for 3ds Max, https://documentation.chaos.com/space/PHX4MAX/124396558, fetched 2026-10-27
