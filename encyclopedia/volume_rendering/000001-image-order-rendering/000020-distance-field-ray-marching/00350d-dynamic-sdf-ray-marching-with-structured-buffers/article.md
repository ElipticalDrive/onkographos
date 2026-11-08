# Dynamic SDF Ray Marching with Structured Buffers

Also called: Procedural Geometry Ray Marching, SDF Ray Marching with Dynamic Object Buffers, SDF Ray Marching with GPU Buffers

## Overview
This method renders dynamic, complex scenes by streaming object definitions from the CPU to the GPU in structured buffers, allowing for per-pixel evaluation of multiple objects without hardcoding the scene geometry in the shader [S1, S2, unofficial]. Unlike static SDF implementations where the scene is hardcoded in GLSL, this approach enables dynamic worlds with many moving characters and objects by replacing CPU-side ray termination logic with a GPU-side buffer of object structs that return a composite type (distance, color, ID) from the distance function [S1, S2, unofficial].

## How it works
**Inputs and Outputs:**
*   **Input:** A subset of scene objects (culled from the full scene), passed as a structured buffer on the GPU. Each element defines the object's type, position, rotation, scale, and average color.
*   **Output:** For each pixel, the final color and depth, derived from the closest object hit by the ray.

**Data Structures:**
1.  **Render Input Buffer (GPU Structured Buffer):**
    *   Layout: Array of `ObjectDefinition` structs.
    *   `ObjectDefinition` fields:
        *   `id`: Integer identifier (inferred from buffer index during search).
        *   `type`: Enum or integer indicating the SDF primitive type (e.g., sphere, box).
        *   `position`: `float3` (world space position).
        *   `rotation`: `float3` or `quat` (orientation).
        *   `scale`: `float3` (uniform or non-uniform scaling).
        *   `color`: `float3` (average/base color).
2.  **Distance Function Return Struct:**
    *   Layout: `struct { float distance; float3 color; int id; }` [S2, unofficial].
    *   `distance`: The local signed distance to the object's surface.
    *   `color`: The color associated with the hit object.
    *   `id`: The ID of the object in the input buffer that produced this distance.

**Numbered Steps:**
1.  **Culling:** Cull the scene into a manageable subset of objects visible in the view frustum. Note that ray-marching performs automatic partial occlusion culling, so aggressive culling may be less critical than in rasterization, but frustum culling is still recommended [S1, S2, unofficial].
2.  **Buffer Preparation:** Pass the subset of objects into a render input buffer on the CPU. Create a typed buffer (e.g., C-style struct) where each element defines the object's function type, position, rotation, scale, and average color [S1, S2, unofficial].
3.  **Upload:** Transfer the buffer to the GPU if it is not already present [S1, S2, unofficial].
4.  **Ray Marching Loop (Per Pixel):**
    *   Initialize ray $r(t) = \mathbf{o} + t\mathbf{d}$ [Parent].
    *   For each iteration $k$:
        *   **Buffer Search:** Perform a per-step search through the input buffer to evaluate which item is closest to the current ray position $\mathbf{p} = \mathbf{o} + t\mathbf{d}$ [S1, S2, unofficial].
        *   **Transform Application:** For each object in the buffer, apply transforms. This can be done by:
            *   Transforming the rays (inverting figure rotations before GPU upload) [S1, S2, unofficial].
            *   Transforming the distance functions (moving function origin for position changes, adjusting side lengths for scale changes) [S1, S2, unofficial].
            *   The simplest approach is to modify the rays before passing them to the core distance function [S1, S2, unofficial].
        *   **SDF Evaluation:** Evaluate the SDF for each object. The distance function returns a complex type (struct) containing the local distance, local color, and an ID tied to the input buffer element [S1, S2, unofficial].
        *   **ID Inference:** The ID is inferred from the loop counter of the buffer search, as the mapping function loops through the input buffer to find the closest figure [S1, S2, unofficial].
        *   **Color Determination:** Determine the color from the average color of the buffer element corresponding to the ID, or procedurally (e.g., mapping iteration count to a palette weighted towards the stored average) [S1, S2, unofficial].
        *   **Step Size:** Use the minimum distance found across all objects in the buffer as the step size for the next iteration [Parent, S1].
        *   **Termination:** If the minimum distance $< \epsilon$ (convergence threshold), the ray has hit the closest object. Terminate the loop [Parent].
5.  **Shading:** Pass the output color directly to the screen or into lighting code based on the collision point and the returned ID/color [S1, S2, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
# Struct definitions (conceptual)
struct ObjectDefinition:
    id: int
    type: int
    position: float3
    rotation: float3 # or quat
    scale: float3
    color: float3

struct SDFResult:
    distance: float
    color: float3
    id: int

# Core SDF for a specific primitive (e.g., sphere)
def sdf_sphere(p: float3, center: float3, radius: float) -> float:
    return length(p - center) - radius

# Generic SDF evaluator for an object
def evaluate_object_sdf(p: float3, obj: ObjectDefinition) -> SDFResult:
    # Apply inverse transform to point p if transforming rays, 
    # or apply transform to p if transforming SDF. 
    # Here we assume transforming the point p to local object space.
    
    # Inverse rotation and translation to get local point
    # Note: Rotation inversion logic depends on representation (quat vs matrix)
    local_p = transform_point_to_local(p, obj.position, obj.rotation)
    
    # Scale adjustment
    local_p = local_p / obj.scale
    
    # Evaluate specific SDF based on type
    dist = 0.0
    if obj.type == SPHERE:
        dist = sdf_sphere(local_p, float3(0), 1.0) # Assuming unit sphere scaled by obj.scale
    elif obj.type == BOX:
        dist = sdf_box(local_p, float3(1)) # Assuming unit box scaled by obj.scale
    else:
        dist = infinity
        
    # Return composite result
    return SDFResult(
        distance=dist,
        color=obj.color,
        id=obj.id
    )

# Main Ray Marching Kernel
def ray_march(ray_origin: float3, ray_direction: float3, objects: Array[ObjectDefinition]) -> float3:
    t = 0.0
    max_steps = 64
    hit_color = float3(0)
    
    for step in range(max_steps):
        current_pos = ray_origin + t * ray_direction
        
        # Per-step search through the input buffer
        min_result = SDFResult(distance=positive_infinity, color=float3(0), id=-1)
        
        for i in range(len(objects)):
            obj = objects[i]
            result = evaluate_object_sdf(current_pos, obj)
            
            # Keep the closest hit
            if result.distance < min_result.distance:
                min_result = result
                # ID is implicitly the loop index i, but stored in result.id by evaluate_object_sdf
        
        # Termination check
        if min_result.distance < 0.001: # epsilon
            hit_color = min_result.color
            break
        elif t > 100.0: # max distance
            break
            
        # Advance ray by the minimum distance found
        t += min_result.distance
        
    return hit_color
```

**Parameters:**
*   **Transform Strategy:** Either transform rays (requires inverting rotations on CPU) or transform distance functions (adjust origin for position, side lengths for scale) [S1, unofficial].
*   **Color Generation:** Can use average color from buffer element or procedural color weighted towards stored average [S1, S2, unofficial].

**Edge Cases and Pitfalls:**
*   **Performance:** The method requires a per-step search through the input buffer for every ray iteration, which can be computationally expensive compared to static scenes [S1, unofficial].
*   **Precision:** Transforming points to local space for SDF evaluation requires careful handling of rotation and scale to avoid precision issues, especially with large scenes or extreme scales.
*   **Buffer Size:** The number of objects in the buffer directly impacts performance. Culling is essential to keep the buffer size manageable.

## Subtopics

### SDF Primitive Definitions
Defines the basic building blocks (spheres, boxes, etc.) used in the buffer. [SDF Primitive Definitions](1578-sdf-primitive-definitions.md)

### Deformation and Transformation Handling
Details on how to apply rotation and scale to objects in the buffer. [Deformation and Transformation Handling](3850-deformation-and-transformation-handling.md)

### Acceleration and Optimization
Techniques to reduce the cost of the per-step buffer search. [Acceleration and Optimization](6222-acceleration-and-optimization.md)

## Operating characteristics
*   **Time Complexity:** $O(N \cdot M)$ per pixel, where $N$ is the number of objects in the buffer and $M$ is the number of ray-marching steps. This is significantly slower than static SDF rendering where $N$ is effectively constant or hardcoded.
*   **Memory Complexity:** $O(N)$ for the structured buffer on the GPU.
*   **Strengths:** Dynamic scenes, easy to add/remove objects, no need to recompile shaders for scene changes.
*   **Limitations:** Performance degrades linearly with the number of objects. Not suitable for very large numbers of objects without additional acceleration structures (e.g., BVH, which is not covered in the basic structured buffer approach).
*   **When to Use:** Real-time rendering of dynamic scenes with a moderate number of objects (e.g., < 1000) where procedural geometry is desired.
*   **When Not to Use:** Scenes with thousands of objects, or where maximum performance is critical and the scene is static.

## Background
This method was described in the context of implementing Signed Distance Field (SDF) ray marching for dynamic worlds, specifically addressing how to stream objects to the GPU and handle multiple colors without hardcoding the scene in GLSL [S1, S2, unofficial]. It is used in procedural geometry engines and real-time SDF rendering, such as in the Claybook engine [S2, unofficial]. The approach leverages GPU structured buffers (or equivalent typed buffers in HLSL/GLSL) to pass complex object data to the shader, allowing the distance function to return a composite type including distance, color, and ID [S1, S2, unofficial].

## Sources
[S1] How is Signed Distance Field Ray Marching implemented for a dynamic world?, https://gamedev.stackexchange.com/questions/111913/how-is-signed-distance-field-ray-marching-implemented-for-a-dynamic-world, fetched 2026-10-26 (unofficial)
[S2] How is Signed Distance Field Ray Marching implemented for a dynamic world?, https://gamedev.stackexchange.com/questions/111913, fetched 2026-10-30 (unofficial)
