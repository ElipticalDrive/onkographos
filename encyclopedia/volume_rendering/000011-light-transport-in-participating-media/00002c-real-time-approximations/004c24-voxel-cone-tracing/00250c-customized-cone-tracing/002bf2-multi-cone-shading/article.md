# Multi-Cone Shading

Also called: Multi-Directional Cone Shading

## Overview
Multi-Cone Shading is a shading technique that samples voxel cone tracing in 5 different directions to approximate diffuse lighting from multiple angles, averaging the results [S1, unofficial]. It builds on [Customized Cone Tracing](3447-customized-cone-tracing.md) by altering diffuse cone behaviors to improve indirect lighting quality [S1, unofficial]. It distinguishes itself from its sibling [Two-Bounce Indirect Illumination](4558-two-bounce-indirect-illumination.md) by focusing on the angular distribution of the initial diffuse bounce rather than simulating secondary bounces, and from [Diffuse Cone Distribution](5677-diffuse-cone-distribution.md) by using a fixed, discrete set of five directions rather than a continuous distribution model [G].

## How it works
The method extends the standard Voxel Cone Tracing algorithm by introducing multi-directional sampling [S1, unofficial]. It operates by defining a local coordinate system aligned with the surface geometry, generating five base cone directions within this local space, rotating them to world space, and accumulating the radiance from each direction [S1, unofficial].

### Inputs and Outputs
*   **Inputs**:
    *   `pp`: The 3D position of the surface point in world space [S1, unofficial].
    *   `direction`: The surface normal vector at `pp` [S1, unofficial].
    *   `coneAngle`: The half-angle of the cone used for tracing, passed to the underlying cone tracing function [S1, unofficial].
    *   `coneRotation`: A uniform float representing the rotation angle in radians, used to offset the cone directions [S1, unofficial].
*   **Outputs**:
    *   `color`: A 4-component vector (RGBA) representing the accumulated diffuse indirect lighting [S1, unofficial].

### Data Structures
No complex data structures are required beyond standard 3D vectors and 3x3 rotation matrices [G]. The core data is the set of five direction vectors [G].

### Steps
1.  **Calculate Trigonometric Values**: Compute the sine and cosine of the `coneRotation` angle [S1, unofficial].
    $$ s = \sin(\text{coneRotation}) $$
    $$ c = \cos(\text{coneRotation}) $$
2.  **Define Base Directions**: Construct five base direction vectors in a local coordinate system where the Z-axis is the primary reference [S1, unofficial].
    $$ \text{dir}_1 = (0, 0, 1) $$
    $$ \text{dir}_2 = (s, 0, c) $$
    $$ \text{dir}_3 = (-s, 0, c) $$
    $$ \text{dir}_4 = (0, s, c) $$
    $$ \text{dir}_5 = (0, -s, c) $$
3.  **Construct Rotation Matrix**: Build a 3x3 rotation matrix `rot` that transforms vectors from the local space to world space, aligning the local Z-axis with the `direction` [S1, unofficial].
    *   Let `zAxis` = `direction` (normalized) [G].
    *   Let `UP` = $(0, 1, 0)$ [S1, unofficial].
    *   Check for gimbal lock: If $|\text{dot}(\text{UP}, \text{zAxis})| < 0.9$, proceed with standard cross products [S1, unofficial]. Otherwise, switch `UP` to $(0, 0, 1)$ to avoid numerical instability [S1, unofficial].
    *   Calculate `xAxis` = $\text{normalize}(\text{cross}(\text{UP}, \text{zAxis}))$ [S1, unofficial].
    *   Calculate `yAxis` = $\text{cross}(\text{zAxis}, \text{xAxis})$ [G].
    *   The rotation matrix `rot` is formed by these basis vectors as columns:
        $$ \text{rot} = \begin{bmatrix} \text{xAxis}_x & \text{yAxis}_x & \text{zAxis}_x \\ \text{xAxis}_y & \text{yAxis}_y & \text{zAxis}_y \\ \text{xAxis}_z & \text{yAxis}_z & \text{zAxis}_z \end{bmatrix} $$
4.  **Transform Directions**: Rotate each base direction into world space [S1, unofficial].
    $$ \text{dir}_i^{\text{world}} = \text{rot} \times \text{dir}_i \quad \text{for } i \in \{1, 2, 3, 4, 5\} $$
5.  **Accumulate Cone Tracing Results**: Initialize an accumulator and cast a cone for each direction using the cone tracing function [S1, unofficial].
    $$ \text{color} = \text{voxelConeTracing}(pp, \text{dir}_1^{\text{world}}, \text{coneAngle}) $$
    $$ \text{color} += \text{voxelConeTracing}(pp, \text{dir}_2^{\text{world}}, \text{coneAngle}) $$
    $$ \text{color} += \text{voxelConeTracing}(pp, \text{dir}_3^{\text{world}}, \text{coneAngle}) $$
    $$ \text{color} += \text{voxelConeTracing}(pp, \text{dir}_4^{\text{world}}, \text{coneAngle}) $$
    $$ \text{color} += \text{voxelConeTracing}(pp, \text{dir}_5^{\text{world}}, \text{coneAngle}) $$
6.  **Average**: Divide the accumulated color by 5 to get the final result [S1, unofficial].
    $$ \text{color} = \text{color} / 5.0 $$
7.  **Return**: Output the averaged `color` [S1, unofficial].

### Pseudocode
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def multi_cone_shading(pp, direction, cone_angle, cone_rotation):
    """
    Approximates diffuse interreflection by casting 5 cones in different directions.
    
    Args:
        pp: vec3, surface position.
        direction: vec3, surface normal.
        cone_angle: float, half-angle of the cone.
        cone_rotation: float, rotation angle in radians.
        
    Returns:
        vec4, accumulated and averaged diffuse color.
    """
    # 1. Calculate sine and cosine of cone rotation
    s = sin(cone_rotation)
    c = cos(cone_rotation)
    
    # 2. Define base directions in local space
    dir1 = vec3(0.0, 0.0, 1.0)
    dir2 = vec3(s, 0.0, c)
    dir3 = vec3(-s, 0.0, c)
    dir4 = vec3(0.0, s, c)
    dir5 = vec3(0.0, -s, c)
    
    # 3. Construct rotation matrix from world normal
    z_axis = normalize(direction)
    up = vec3(0.0, 1.0, 0.0)
    
    # Handle gimbal lock
    if abs(dot(up, z_axis)) < 0.9:
        x_axis = normalize(cross(up, z_axis))
        y_axis = cross(z_axis, x_axis)
    else:
        up = vec3(0.0, 0.0, 1.0)
        x_axis = normalize(cross(up, z_axis))
        y_axis = cross(z_axis, x_axis)
        
    # Build rotation matrix (columns are x, y, z axes)
    rot = mat3(x_axis, y_axis, z_axis)
    
    # 4. Transform base directions to world space
    w_dir1 = rot * dir1
    w_dir2 = rot * dir2
    w_dir3 = rot * dir3
    w_dir4 = rot * dir4
    w_dir5 = rot * dir5
    
    # 5. Cast cones and accumulate
    # voxel_cone_tracing is defined in [Customized Cone Tracing](3447-customized-cone-tracing.md). Not given in the sources.
    color = voxel_cone_tracing(pp, w_dir1, cone_angle)
    color += voxel_cone_tracing(pp, w_dir2, cone_angle)
    color += voxel_cone_tracing(pp, w_dir3, cone_angle)
    color += voxel_cone_tracing(pp, w_dir4, cone_angle)
    color += voxel_cone_tracing(pp, w_dir5, cone_angle)
    
    # 6. Average
    color = color / 5.0
    
    return color

def voxel_cone_tracing(pp, direction, cone_angle):
    # Implementation defined in [Customized Cone Tracing](3447-customized-cone-tracing.md). Not given in the sources.
    return vec4(0.0)
```

### Parameters and Typical Values
*   `coneAngle`: Controlled by the user or derived from the voxel grid resolution [S1, unofficial]. Typical values: Not given in the sources. [S1, unofficial]
*   `coneRotation`: A uniform float representing the rotation angle in radians [S1, unofficial]. Typical values: Not given in the sources. [S1, unofficial]
*   `pp`: Varies per pixel [S1, unofficial].
*   `direction`: Varies per pixel (surface normal) [S1, unofficial].

### Edge Cases and Pitfalls
*   **Gimbal Lock**: When the surface normal is parallel to the up vector, the cross product for the X-axis becomes zero [G]. The algorithm explicitly checks $|\text{dot}(\text{UP}, \text{zAxis})| < 0.9$ and switches the up vector to $(0,0,1)$ to maintain numerical stability [S1, unofficial].
*   **Precision**: Floating-point precision errors in the rotation matrix construction can lead to non-orthogonal basis vectors [G]. Normalization of axes is critical [G].
*   **Boundary Conditions**: If the surface point is outside the voxel grid, the underlying cone tracing function should return black or zero [G]. Not given in the sources. [S1, unofficial]

## Subtopics

### [Two-Bounce Indirect Illumination](4558-two-bounce-indirect-illumination.md)
Simulates secondary bounces of light. Choose when global illumination depth is more important than angular resolution of the first bounce [G].

### [Diffuse Cone Distribution](5677-diffuse-cone-distribution.md)
Uses a continuous distribution of cone directions. Choose when smoother angular sampling is needed and performance allows for more complex integration [G].

## Operating characteristics
*   **Time Complexity**: Not given in the sources. [S1, unofficial]
*   **Memory Complexity**: Not given in the sources. [S1, unofficial]
*   **Performance**: Measured performance depends heavily on the underlying cone tracing implementation [G]. Not given in the sources. [S1, unofficial]
*   **Strengths**: Significantly improves the visual quality of diffuse interreflection compared to single-direction tracing by reducing shadowing artifacts and providing a more isotropic appearance [G].
*   **Limitations**: 5x the computational cost of single-direction tracing [G]. Still an approximation; does not capture complex indirect lighting paths beyond the first bounce [G].
*   **When to use**: When high-quality diffuse interreflection is required in real-time applications and single-direction cone tracing produces overly dark or shadowed results [G].
*   **When not to use**: When performance is critical and single-direction tracing is sufficient, or when full global illumination is required [G].

## Background
Multi-Cone Shading is an informal technique implemented in the open-source project [WebGPUVoxelConeTracing](https://github.com/HectorArellanoDev/WebGPUVoxelConeTracing) by Hector Arellano [S1, unofficial]. It builds upon the principles of Voxel Cone Tracing, a technique popularized by Matt Pettineo for real-time global illumination [G]. The specific multi-directional approach is a heuristic to improve the approximation of the diffuse integral without resorting to more expensive methods like spherical harmonics or light propagation volumes [G]. It is not attributed to a specific peer-reviewed paper in the provided sources. [S1, unofficial]

## Sources
[S1] HectorArellanoDev/WebGPUVoxelConeTracing, https://github.com/HectorArellanoDev/WebGPUVoxelConeTracing, fetched 2026-10-26 (unofficial)
