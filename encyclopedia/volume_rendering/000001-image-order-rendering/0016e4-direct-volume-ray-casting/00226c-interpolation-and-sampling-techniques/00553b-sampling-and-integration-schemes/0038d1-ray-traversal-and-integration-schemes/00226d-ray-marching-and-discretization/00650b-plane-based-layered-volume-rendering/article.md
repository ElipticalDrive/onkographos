# Plane-Based Layered Volume Rendering

Also called: Discrete Plane Volume Sampling, Exclusion Plane Volume Rendering

## Overview
Plane-Based Layered Volume Rendering is a direct volume ray casting technique that approximates continuous volume integration by sampling a discrete set of parallel planes aligned with the camera view direction [S1, unofficial]. It differs from standard fixed-step ray marching by restricting samples to specific geometric planes, which allows for optional exclusion plane testing to skip empty or occluded regions, thereby reducing unnecessary texture lookups and shading computations [S1, unofficial].

## How it works
The method renders a volume by iterating through a predefined number of layers (planes). For each pixel, a ray is cast from the camera position. The algorithm calculates the intersection of this ray with each layer plane, transforms the intersection point into local volume coordinates, and samples the volume texture. An optional exclusion test can skip sampling if the point lies outside defined exclusion planes. The resulting color and opacity are accumulated using front-to-back alpha blending.

### Inputs and Outputs
**Inputs:**
*   `MODEL_MATRIX`: 4x4 transformation matrix from local to world space [S1, unofficial].
*   `CAMERA_POSITION_WORLD`: 3D vector representing the camera's position in world space [S1, unofficial].
*   `pos_world`: 3D vector representing the current ray position or target point in world space [S1, unofficial].
*   `texture_volume`: 3D texture containing the volume data (RGBA8 format implied by source) [S1, unofficial].
*   `gradient`: 1D or 2D texture mapping scalar values to color/opacity (used for transfer function) [S1, unofficial].
*   `layers`: Integer, total number of planes to sample [S1, unofficial].
*   `zoom`: Float, scaling factor applied to local coordinates before sampling [S1, unofficial].
*   `opacity`: Float, global opacity multiplier [S1, unofficial].
*   `color_scalar`: Float, global color intensity multiplier [S1, unofficial].
*   `gamma`: Float, gamma correction exponent applied to the scalar value before gradient lookup [S1, unofficial].
*   `exclusion_planes`: Array of `vec4` plane equations $(A, B, C, D)$ where $Ax + By + Cz + D > 0$ indicates exclusion [S1, unofficial].
*   `MAX_EXCLUSION_PLANES`: Integer, maximum number of exclusion planes supported [S1, unofficial].
*   `num_exclusion_planes`: Integer, actual number of exclusion planes provided [S1, unofficial].

**Outputs:**
*   `col`: Accumulated `vec4` color output for the pixel [S1, unofficial].

### Data Structures
*   **Exclusion Plane**: Represented as a `vec4` $(A, B, C, D)$ defining the plane equation $Ax + By + Cz + D = 0$. A point $P$ is excluded if $dot(vec4(P, -1.0), plane) > 0.0$ [S1, unofficial]. Note: The source uses $-1.0$ for the w-component, implying the plane equation is evaluated as $Ax + By + Cz - D > 0$ or the plane vector stores $-D$ in the w-component. Based on standard homogeneous coordinates $Ax+By+Cz+Dw$, if $w=-1$, the term is $-D$. Thus, the stored `vec4` likely contains $(A, B, C, -D_{standard})$ or the logic explicitly checks against $-D$. The code `dot(vec4(p_world, -1.0), exclusion_planes[i])` expands to $A x + B y + C z + (-1) D_{stored}$. If this is $>0$, it is excluded.

### Parameters
*   `layers`: Not given in the sources. Typical values depend on volume density and desired accuracy.
*   `zoom`: Not given in the sources. Used to scale local coordinates.
*   `opacity`: Not given in the sources.
*   `color_scalar`: Not given in the sources.
*   `gamma`: Not given in the sources.

### Algorithm Steps
1.  **Calculate Plane Normal**: Determine the direction from the volume model origin to the camera.
    $$ \mathbf{n} = \text{normalize}(\mathbf{C}_{world} - \mathbf{O}_{model\_world}) $$
    Where $\mathbf{C}_{world}$ is `CAMERA_POSITION_WORLD` and $\mathbf{O}_{model\_world}$ is `MODEL_MATRIX[3].xyz` [S1, unofficial].

2.  **Iterate Layers**: Loop from $i = 0$ to $layers - 1$.
    a.  **Calculate Plane Position**: Determine the origin of the current layer plane in world space.
        $$ \mathbf{P}_{plane\_origin} = \mathbf{O}_{model\_world} + \mathbf{n} \cdot \left( \frac{i + 1}{layers + 1} \cdot 2 - 1 \right) $$
        This formula centers the planes within the volume extent defined by the model origin and the normal direction, distributing them evenly between $-1$ and $1$ relative to the model origin along the normal [S1, unofficial].
    b.  **Ray-Plane Intersection**: Compute the intersection point $\mathbf{P}_{world}$ of the ray (origin $\mathbf{C}_{world}$, direction $\mathbf{d} = \mathbf{P}_{world\_target} - \mathbf{C}_{world}$) with the plane defined by $\mathbf{P}_{plane\_origin}$ and normal $\mathbf{n}$.
        $$ \mathbf{P}_{world} = \text{isect\_plane}(\mathbf{C}_{world}, \mathbf{d}, \mathbf{P}_{plane\_origin}, \mathbf{n}) $$
        [S1, unofficial].
    c.  **Exclusion Test**: Check if $\mathbf{P}_{world}$ lies in any exclusion region.
        $$ \text{excluded} = \bigvee_{j=0}^{\min(num\_exclusion\_planes, MAX\_EXCLUSION\_PLANES)-1} \left( \text{dot}(\text{vec4}(\mathbf{P}_{world}, -1.0), \mathbf{E}_j) > 0.0 \right) $$
        If excluded, skip to the next layer [S1, unofficial].
    d.  **Transform to Local Space**: Convert $\mathbf{P}_{world}$ to local volume coordinates.
        $$ \mathbf{P}_{local} = \text{xyz}(\text{inverse}(\text{MODEL\_MATRIX}) \cdot \text{vec4}(\mathbf{P}_{world}, 1.0)) $$
        [S1, unofficial].
    e.  **Texture Coordinate Calculation**: Map local coordinates to texture space $[0, 1]^3$.
        $$ \mathbf{T} = \frac{\mathbf{P}_{local} \cdot \text{zoom}}{2.0} + \text{vec3}(0.5) $$
        [S1, unofficial].
    f.  **Volume Sample**: Sample the volume texture.
        $$ \mathbf{C}_{layer} = \text{texture}(\text{texture\_volume}, \mathbf{T}) $$
        [S1, unofficial].
    g.  **Transfer Function**: Map the scalar value (red channel) to color/opacity using the gradient texture.
        $$ \mathbf{C}_{layer} = \text{texture}(\text{gradient}, \text{vec2}(\mathbf{C}_{layer}.r^\gamma, 0)) $$
        [S1, unofficial].
    h.  **Apply Opacity and Color**:
        $$ \mathbf{C}_{layer}.a \leftarrow \mathbf{C}_{layer}.a \cdot \text{opacity} $$
        $$ \mathbf{C}_{layer} \leftarrow \mathbf{C}_{layer} \cdot \text{color\_scalar} $$
        [S1, unofficial].
    i.  **Alpha Blending**: Accumulate the layer color into the result.
        $$ \mathbf{C}_{result} = \text{mix}(\mathbf{C}_{result}, \mathbf{C}_{layer}, \mathbf{C}_{layer}.a) $$
        [S1, unofficial].

### Pseudocode
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def plane_based_layered_volume_rendering(
    camera_pos_world: vec3,
    ray_direction: vec3, # Or target point to derive direction
    model_matrix: mat4,
    texture_volume: texture3D,
    gradient_texture: texture1D,
    layers: int,
    zoom: float,
    opacity: float,
    color_scalar: float,
    gamma: float,
    exclusion_planes: list[vec4],
    max_exclusion_planes: int
) -> vec4:
    model_origin_world = model_matrix[3].xyz
    plane_norm = normalize(camera_pos_world - model_origin_world)
    
    result_color = vec4(0.0, 0.0, 0.0, 0.0)
    
    # Determine ray origin and direction for intersection
    # Note: The source uses 'pos_world' in isect_plane, implying the ray 
    # might be defined by camera_pos and a target point, or the function 
    # signature in the source implies 'pos_world' is the point being tested 
    # against the plane. However, standard ray casting requires a ray.
    # The source code: `isect_plane(CAMERA_POSITION_WORLD, pos_world - CAMERA_POSITION_WORLD, ...)`
    # This implies the ray is: Origin = CAMERA_POSITION_WORLD, Direction = pos_world - CAMERA_POSITION_WORLD.
    # 'pos_world' here likely refers to the fragment position or a target point in world space 
    # associated with the current pixel being rendered.
    
    ray_origin = camera_pos_world
    ray_direction = ray_direction # Assumed to be passed or derived from screen coords
    
    inv_model_mtx = inverse(model_matrix)
    
    for layer_idx in range(layers):
        layer_frac = (layer_idx + 1) / (layers + 1)
        plane_origin = model_origin_world + plane_norm * (layer_frac * 2.0 - 1.0)
        
        # Calculate intersection of ray with plane
        # isect_plane(origin, direction, plane_origin, plane_normal)
        # Standard formula: t = dot(plane_normal, plane_origin - origin) / dot(plane_normal, direction)
        # P = origin + t * direction
        denom = dot(plane_norm, ray_direction)
        if abs(denom) < 1e-6:
            continue # Ray parallel to plane
            
        t = dot(plane_norm, plane_origin - ray_origin) / denom
        p_world = ray_origin + t * ray_direction
        
        # Exclusion test
        plane_test_passed = True
        num_checks = min(len(exclusion_planes), max_exclusion_planes)
        for i in range(num_checks):
            # dot(vec4(p_world, -1.0), exclusion_planes[i])
            # This is equivalent to: A*x + B*y + C*z + D*(-1) > 0
            # If exclusion_planes[i] is (A, B, C, D), then check A*x + B*y + C*z - D > 0
            val = dot(vec4(p_world.x, p_world.y, p_world.z, -1.0), exclusion_planes[i])
            if val > 0.0:
                plane_test_passed = False
                break
        
        if not plane_test_passed:
            continue
            
        # Transform to local space
        p_local = (inv_model_mtx * vec4(p_world, 1.0)).xyz
        
        # Calculate texture coordinates
        tex_sample_pt = (p_local * zoom) / 2.0 + vec3(0.5, 0.5, 0.5)
        
        # Sample volume
        layer_col = texture(texture_volume, tex_sample_pt)
        
        # Apply transfer function
        # Note: source uses vec2(pow(layer_col.r, gamma), 0)
        # This implies a 2D texture lookup where the second coordinate is 0, 
        # effectively using a 1D gradient mapped to the first coordinate.
        layer_col = texture(gradient_texture, vec2(pow(layer_col.r, gamma), 0.0))
        
        # Apply opacity and color scalar
        layer_col.a *= opacity
        layer_col *= color_scalar
        
        # Front-to-back blending
        # mix(base, color, alpha) -> base * (1 - alpha) + color * alpha
        result_color = mix(result_color, layer_col, layer_col.a)
        
    return result_color
```

### Edge Cases and Pitfalls
*   **Parallel Ray**: If the ray direction is parallel to the plane normal (dot product near zero), the intersection calculation fails. A guard against division by zero is required [G].
*   **Exclusion Plane W-component**: The source uses `vec4(p_world, -1.0)`. This is non-standard for homogeneous plane equations ($Ax+By+Cz+Dw$). Implementers must ensure the exclusion plane data is stored such that the dot product with $(x, y, z, -1)$ correctly represents the exclusion condition. If standard planes $(A,B,C,D)$ are used, the check should be $Ax+By+Cz-D > 0$ or the plane data should store $-D$ in the w-component [S1, unofficial].
*   **Texture Coordinate Bounds**: The formula `(p_local * zoom) / 2.0 + 0.5` assumes `p_local` is within $[-1, 1]$ when `zoom` is 1. If `zoom` is large or `p_local` is out of bounds, texture sampling may return undefined values or clamp depending on sampler settings. Clamping or border color handling should be considered [G].
*   **Gamma Application**: Gamma is applied to the red channel before the gradient lookup. This assumes the volume data's red channel represents the scalar density/value [S1, unofficial].

## Operating characteristics
*   **Time Complexity**: $O(L)$ per pixel, where $L$ is the number of layers. Each layer involves a ray-plane intersection, a matrix multiplication, and texture lookups. Exclusion tests add $O(E)$ per layer, where $E$ is the number of exclusion planes.
*   **Memory Complexity**: $O(V)$ for the volume texture and $O(E)$ for exclusion planes.
*   **Strengths**: Reduces the number of texture samples compared to dense fixed-step marching by skipping empty regions via exclusion planes. Aligns with the view direction, which can be efficient for certain data structures.
*   **Limitations**: The discrete nature of planes may introduce aliasing or artifacts if the layer spacing is too large. The method is view-dependent; the planes are aligned with the camera, so it does not naturally support view-independent optimizations like spatial partitioning for all views.
*   **When to use**: Suitable for volumes with large empty regions that can be defined by exclusion planes, or when a specific layered approximation is desired for performance.
*   **When not to use**: For volumes requiring high accuracy without gaps, or when the volume data is not well-aligned with the view direction and exclusion planes are not effective.

## Background
The method is described in the context of the `godot_volume_layers` addon, an unofficial implementation for the Godot engine [S1, unofficial]. It represents a specific approach to volume rendering within that ecosystem, leveraging compute shaders for mipmap generation and fragment shaders for the layered sampling.

## Sources
[S1] blackears/godot_volume_layers, https://github.com/blackears/godot_volume_layers, fetched 2026-10-26 (unofficial)
