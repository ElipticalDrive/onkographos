# Ray-Box Intersection via Quad Rasterization


## Overview
A voxel rendering technique where each voxel is represented by a single point in a vertex array, sized to cover the voxel's projected area, with the actual voxel geometry (cube) intersected with a ray in the fragment shader using an optimized ray-box intersection test [S1, unofficial]. It sets itself apart from sibling methods like [Sparse Voxel Octree Rendering via Square Splats](2993-sparse-voxel-octree-rendering-via-square-splats.md) and [Voxel Point Sprite Rendering](4307-voxel-point-sprite-rendering.md) by deferring the geometric intersection logic entirely to the fragment shader, allowing for a single draw call per voxel set rather than rasterizing explicit cube faces or traversing tree structures [S1, unofficial].

## How it works
**Inputs and Outputs:**
- Input: A set of voxel positions encoded as 32-bit integers in a vertex buffer, camera parameters (position, projection matrix), and screen resolution.
- Output: A depth-buffered image where voxels are rendered as solid cubes.

**Data Structures:**
The vertex buffer stores one entry per voxel. Each entry consists of two 32-bit unsigned integers:
1. `pos`: A packed 32-bit integer representing the voxel's $(x, y, z)$ coordinates.
   - Bits 0-10: $x$ coordinate (11 bits).
   - Bits 11-16: $y$ coordinate (6 bits, shifted left by 11).
   - Bits 17-31: $z$ coordinate (15 bits, shifted left by 21).
2. `albedo`: A packed 32-bit integer representing color.
   - Bits 0-4: Red (5 bits).
   - Bits 5-10: Green (6 bits, shifted left by 5).
   - Bits 11-15: Blue (5 bits, shifted left by 11).
   - Bits 16-31: Alpha (16 bits, unused in the provided snippet but part of the 32-bit word).

In the fragment shader, the following structures are used for intersection:
- `Box`: Contains `vec3 center`, `vec3 radius`, `vec3 invRadius`, and `mat3 rotation`.
- `Ray`: Contains `vec3 origin` and `vec3 direction`.

**Numbered Steps:**
1. **Vertex Setup**: Create a vertex buffer containing one vertex per voxel. The vertex position is typically $(0,0,0)$ in local space, but the point size is set to cover the projected area of the voxel cube. Enable `GL_PROGRAM_POINT_SIZE` to allow the vertex shader to define the point size dynamically [S1, unofficial].
2. **Draw Call**: Render all voxels using a single `glDrawArrays` call with the `GL_POINTS` primitive [S1, unofficial].
3. **Fragment Shader - Ray Reconstruction**:
   - Reconstruct the eye-space position from `gl_FragCoord`.
   - Compute the ray direction as the normalized vector from the reconstructed eye-space position to the camera origin:
     $$ray.direction = -\frac{camera\_origin - eyePos}{||camera\_origin - eyePos||}$$
   - Set `ray.origin` to `camera_origin` [S1, unofficial].
4. **Fragment Shader - Intersection Test**:
   - Define the voxel box. The voxel center is derived from the vertex data (unpacked from `pos`). The radius is hardcoded to $0.05$ in the fragment shader [S1, unofficial].
   - Call the intersection function `ourIntersectBoxCommon` with the voxel center, radius, and ray. Pass `false` for `rayCanStartInBox` and `false` for `oriented` to optimize for axis-aligned boxes where the ray starts outside [S1, unofficial].
5. **Fragment Shader - Depth Writing**:
   - If the intersection test fails, discard the fragment.
   - If it succeeds, compute the fragment depth using the intersection distance and write it to `gl_FragDepth` to ensure correct depth testing [S1, unofficial].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
# Vertex Shader (GLSL-like)
# Input: vertex index i
# Output: gl_Position, gl_PointSize

void main() {
    // Unpack position from vertex buffer if necessary, 
    # but typically points are rendered at (0,0,0) and transformed by model/view/projection.
    # The key is setting the point size to cover the voxel.
    gl_Position = projection_matrix * view_matrix * model_matrix * vec4(0.0, 0.0, 0.0, 1.0);
    gl_PointSize = point_size_covering_voxel; # Set to cover projected cube area
}

# Fragment Shader (GLSL-like)
# Input: gl_FragCoord, vertex attributes (pos, albedo)
# Output: gl_FragColor, gl_FragDepth

struct Ray {
    vec3 origin;
    vec3 direction;
};

struct Box {
    vec3 center;
    vec3 radius;
    vec3 invRadius;
    mat3 rotation;
};

// Optimized ray-box intersection for axis-aligned, ray outside box
bool ourIntersectBoxCommon(Ray ray, Box box, out float tmin, out float tmax) {
    // Precompute invRayDirection to avoid division by zero in shader
    vec3 invRayDir = 1.0 / ray.direction;
    
    vec3 t0 = (box.center - box.radius - ray.origin) * invRayDir;
    vec3 t1 = (box.center + box.radius - ray.origin) * invRayDir;
    
    vec3 tmin_vec = min(t0, t1);
    vec3 tmax_vec = max(t0, t1);
    
    tmin = max(max(tmin_vec.x, tmin_vec.y), tmin_vec.z);
    tmax = min(min(tmax_vec.x, tmax_vec.y), tmax_vec.z);
    
    return tmax >= tmin;
}

void main() {
    // Reconstruct eye-space position
    vec4 clip_pos = vec4(gl_FragCoord.xy, 0.0, 1.0); // z=0 for near plane approximation or use inverse projection
    // More accurate reconstruction:
    vec4 ndc = vec4(gl_FragCoord.x / viewport_width * 2.0 - 1.0, 
                    gl_FragCoord.y / viewport_height * 2.0 - 1.0, 
                    0.0, 1.0);
    vec4 eye_pos = inverse_projection_matrix * ndc;
    eye_pos /= eye_pos.w;
    
    vec3 ray_origin = camera_position;
    vec3 ray_direction = normalize(eye_pos.xyz - camera_position);
    
    // Unpack voxel center from vertex attribute
    uint pos = vertex_pos;
    uint x = pos & 0x7FF;
    uint y = (pos >> 11) & 0x3F;
    uint z = (pos >> 17) & 0x7FFF;
    vec3 voxel_center = vec3(float(x), float(y), float(z));
    
    Box box;
    box.center = voxel_center;
    box.radius = vec3(0.05); // Hardcoded radius
    box.invRadius = 1.0 / box.radius;
    box.rotation = mat3(1.0); // Axis-aligned
    
    Ray ray;
    ray.origin = ray_origin;
    ray.direction = ray_direction;
    
    float tmin, tmax;
    if (!ourIntersectBoxCommon(ray, box, tmin, tmax)) {
        discard;
    }
    
    // Compute depth
    float depth = length(ray.origin + ray.direction * tmin - camera_position);
    gl_FragDepth = depth;
    
    // Output color
    gl_FragColor = vec4(vertex_albedo / 255.0, 1.0);
}
```

**Parameters:**
- Voxel radius: $0.05$ (hardcoded in the fragment shader) [S1, unofficial].
- Point size: Sized to cover the voxel's projected area [S1, unofficial].

**Edge Cases and Pitfalls:**
- **Division by Zero**: When computing distance to plane, if the ray direction component is zero, division by zero occurs. The code precomputes `invRayDirection` ($1.0 / ray.direction$) and passes it to avoid runtime division, assuming finite values [S1, unofficial].
- **Ray Origin Inside Box**: The intersection function is called with `rayCanStartInBox = false`, which is an optimization for axis-aligned boxes where the ray starts outside. If the ray starts inside, this may lead to incorrect intersection results [S1, unofficial].

## Subtopics

### [Sparse Voxel Octree Rendering via Square Splats](2993-sparse-voxel-octree-rendering-via-square-splats.md)
Renders voxels as square splats but traverses a sparse voxel octree structure. Choose when memory efficiency is critical and the scene is sparse.

### [Voxel Point Sprite Rendering](4307-voxel-point-sprite-rendering.md)
Renders voxels as point sprites with depth peeling or similar techniques for transparency. Choose when handling transparent voxels is required.

## Operating characteristics
- **Time Complexity**: $O(N)$ where $N$ is the number of voxels, as each voxel is processed in a single draw call. The fragment shader performs a constant-time ray-box intersection.
- **Memory Complexity**: $O(N)$ for the vertex buffer storing voxel positions and attributes.
- **Measured Performance**: On dedicated graphics cards, MODE_QUAD is approximately twice as fast as MODE_CUBE. On Intel Integrated graphics, performance is similar between the two techniques [S1, unofficial].
- **Strengths**: Single draw call for all voxels, efficient use of fragment shader for intersection, reduced overdraw compared to MODE_CUBE.
- **Limitations**: Requires fragment shader support for ray-box intersection, potential precision issues with ray reconstruction, not suitable for highly transparent scenes without additional techniques.
- **When to Use**: Dense voxel scenes on dedicated GPUs where performance is critical.
- **When Not to Use**: Scenes with high transparency requiring depth peeling, or on integrated graphics where the performance gain is negligible.

## Background
Originated from a short test project in efficient voxel rendering by Clay John, demonstrating two modes: MODE_QUAD and MODE_CUBE. MODE_QUAD uses the ray-box intersection method described in the Journal of Computer Graphics Techniques (JCGT) paper "Ray-Box Intersection for 3D Grids" (https://jcgt.org/published/0007/03/04/). The technique was developed as an experiment in high-performance voxel rendering, leveraging modern GPU capabilities for fragment shader-based ray tracing [S1, unofficial].

## Sources
[S1] GitHub - clayjohn/voxel-renderer: A short experiment in high ..., https://github.com/clayjohn/voxel-renderer, fetched 2026-10-27 (unofficial)
