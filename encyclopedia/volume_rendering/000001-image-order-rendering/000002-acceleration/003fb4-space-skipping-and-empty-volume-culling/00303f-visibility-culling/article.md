# Visibility Culling
Also called: Ray-guided volume rendering, Implicit visibility culling
## Overview
Visibility Culling is a modern acceleration technique for large-scale volume rendering that determines the working set of data (bricks or cells) directly during ray traversal, eliminating the need for explicit pre-passes for view frustum or occlusion culling [S1, unofficial]. It solves the problem of redundant data loading and processing by allowing rays to write out the list of bricks they intersect in front-to-back order, effectively culling invisible geometry without extra steps or occlusion buffers [S1, unofficial]. Unlike traditional methods that require separate culling stages or complex data structures to manage visibility, this approach integrates visibility determination into the rendering pipeline itself, leveraging modern GPU extensions for efficient data management [S1, unofficial].

## How it works
Visibility Culling operates by having the ray traversal logic directly identify and request the necessary data volumes (bricks) for each pixel's ray. This process replaces explicit culling passes with implicit determination based on ray intersection.

### Inputs and Outputs
- **Input**: Ray origins and directions for each pixel; volume data organized into bricks (or cells); a shared cache texture (brick pool) accessible by the shader [S1, unofficial].
- **Output**: A list of bricks required for rendering, written out in front-to-back order; final pixel colors after compositing [S1, unofficial].

### Data Structures
- **Shared Cache Texture ("Brick Pool")**: A texture that stores all available bricks. This allows the GPU to manage paging and caching (e.g., LRU, MRU) automatically or via manual strategies if the working set exceeds capacity [S1, unofficial].
- **Ray-Generated Brick List**: Rays write the identifiers of the bricks they intersect. The order is front-to-back to support correct compositing [S1, unofficial].

### Steps
1. **Ray Initialization**: For each pixel, initialize a ray with the camera origin and direction [S1, unofficial].
2. **Traversal and Intersection**: Traverse the volume structure (e.g., octree or grid) to find intersected bricks. This step implicitly handles view frustum culling because rays outside the volume do not intersect any bricks [S1, unofficial].
3. **Brick Identification**: As the ray intersects bricks, record their identifiers. This list represents the working set for that ray [S1, unofficial].
4. **Data Fetching**: Use the brick identifiers to fetch the corresponding voxel data from the shared cache texture. Modern OpenGL extensions such as `GL_ARB_shader_storage_buffer_object` are used to facilitate efficient writing of the brick list and reading from the cache [S1, unofficial].
5. **Compositing**: Perform front-to-back compositing using the fetched data. Early termination can be applied here if transmittance drops below a threshold [S1, S6].
6. **Implicit Occlusion Culling**: Because the ray traversal only accesses bricks along the ray path, occluded bricks (those behind opaque surfaces) are naturally skipped, providing implicit occlusion culling without additional buffers [S1, unofficial].

### Formulas
- **Ray Equation**: $R(t) = O + tD$, where $O$ is the origin, $D$ is the direction, and $t$ is the distance [S1, unofficial].
- **Transmittance Update**: $T(t + \Delta t) = T(t) \cdot e^{-\sigma_t \Delta t}$, where $\sigma_t$ is the extinction coefficient [S1, S6].

### Parameters
- **Brick Size**: Not given in the sources.
- **Cache Policy**: LRU (Least Recently Used) or MRU (Most Recently Used) for managing the shared cache texture [S1, unofficial].

### Edge Cases and Pitfalls
- **Working Set Overflow**: If the working set is too large to fit in the shared cache, strategies such as switching to multi-pass rendering or interrupting rendering must be employed [S1, unofficial].
- **Precision**: Ray traversal precision affects the accuracy of brick intersection. Insufficient precision may lead to missing bricks or accessing incorrect data [G].

### Pseudocode
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
# Pseudocode for Visibility Culling in a fragment shader
# Assumes a shared cache texture 'brick_pool' and a mechanism to write brick IDs

def visibility_culling(ray_origin, ray_direction, brick_pool):
    """
    Performs visibility culling by traversing the volume and fetching bricks.
    """
    # Initialize transmittance and color
    transmittance = 1.0
    color = vec3(0.0)
    
    # Ray traversal parameters
    t = 0.0
    step_size = get_step_size() # Depends on resolution and brick size
    
    # List to store intersected bricks (front-to-back)
    intersected_bricks = []
    
    # Traverse the volume
    while t < max_distance:
        # Determine current brick ID based on ray position
        # This step implicitly handles frustum and occlusion culling
        brick_id = get_brick_id_at(ray_origin + ray_direction * t)
        
        # If brick is valid and not already processed
        if brick_id is not None and brick_id not in intersected_bricks:
            intersected_bricks.append(brick_id)
            
            # Fetch brick data from shared cache
            brick_data = fetch_brick_data(brick_id, brick_pool)
            
            # Sample and composite
            sample = sample_brick(brick_data, ray_origin + ray_direction * t)
            alpha = sample.opacity
            density = sample.density
            
            # Update transmittance and color
            transmittance *= exp(-density * step_size)
            color += sample.color * transmittance * step_size
            
            # Early termination if fully opaque
            if transmittance < 0.01:
                break
        
        t += step_size
    
    return color
```

## Subtopics

### [Occlusion Clipping with Orthogonal Opacity Map](990-occlusion-clipping-with-orthogonal-opacity-map.md)
Occlusion Clipping with Orthogonal Opacity Map is an acceleration technique for texture-based volume rendering that skips occluded voxels by maintaining a low-resolution 2D opacity map on a plane orthogonal to a major dataset axis. It differs from per-voxel early ray termination by testing occlusion at the sub-volume level, allowing it to cull entire bounding boxes or clip their geometry against the opacity map to avoid rendering thousands of voxels hidden behind opaque surfaces.

### [Early Termination and Probabilistic Path Control](1791-early-termination-and-probabilistic-path-control.md)
Early Termination and Probabilistic Path Control are visibility culling techniques that reduce computational cost by halting ray traversal when further samples contribute negligibly to the final pixel color. Unlike Empty-space skipping, which bypasses transparent regions entirely, these methods actively sample until a radiometric threshold (opacity or transmittance) is met, or probabilistically terminate paths to maintain unbiasedness. They differ from Bounding Volume Culling by using material properties rather than geometric extent to decide when to stop.

### [Probabilistic Culling](2833-probabilistic-culling.md)
Probabilistic Culling is a rendering optimization that intentionally allows false-negatives in intersection tests to reduce duplicated fragments and rendering time, particularly in high-resolution or virtual reality contexts. It sets itself apart from conservative culling methods (like Occlusion Clipping with Orthogonal Opacity Map) by trading strict geometric accuracy for performance, relying on the fact that the resulting visual artifacts are imperceptible to the human eye.

### [Visibility Feedback with Screen Space Subdivision](3439-visibility-feedback-with-screen-space-subdivision.md)
Visibility Feedback with Screen Space Subdivision is an out-of-core volume rendering optimization that uses visibility information gathered during the current frame to prevent the loading and refinement of occluded data in subsequent frames. Unlike standard early ray termination, which only skips computation after data has already been loaded into the GPU working set, this method feeds visibility results back to the loader to reduce GPU memory usage and bandwidth. It distinguishes itself from sibling methods by combining occlusion queries with a screen-space subdivision strategy to interleave query processing with rasterization, thereby hiding the latency inherent in asynchronous occlusion query results.

### [Adaptive Multi-Step Empty Voxel Skipping](6289-adaptive-multi-step-empty-voxel-skipping.md)
Adaptive Multi-Step Empty Voxel Skipping is an acceleration technique for direct volume rendering that combines space leaping (empty voxel skipping) with adaptive variable step-length sampling to reduce redundant computation in sparse regions. It distinguishes itself from standard ray casting by dynamically adjusting the integration step size based on local voxel density and explicitly bypassing empty space, whereas siblings like Occlusion Clipping with Orthogonal Opacity Map rely on precomputed opacity maps and Early Termination and Probabilistic Path Control rely on transmittance thresholds for termination rather than spatial skipping.

### [24-Region View-Dependent Visibility Preprocessing](6608-24-region-view-dependent-visibility-preprocessing.md)
A preprocessing technique that decomposes the domain of viewing directions into 24 regions on a directional cube to extract a lower number of potentially visible voxels compared to the standard 6-region decomposition, improving extraction rate and rendering speed. It sets itself apart from Occlusion Clipping with Orthogonal Opacity Map by using a directional decomposition strategy rather than orthogonal projection maps, and from Adaptive Multi-Step Empty Voxel Skipping by focusing on surface boundary extraction rather than empty space traversal.

### [Frustum Culling](8463-frustum-culling.md)
Frustum culling is a visibility culling technique that reduces rendering load by identifying and excluding geometry, chunks, or voxels that lie outside the camera's view frustum. It solves the problem of unnecessary computation for primitives that do not contribute to the final image. Unlike Occlusion Clipping with Orthogonal Opacity Map, which handles depth-based occlusion, or Early Termination and Probabilistic Path Control, which handles ray integration termination, frustum culling operates on spatial bounds to reject entire regions before they enter the rendering pipeline. It is often combined with Proximity Culling to manage data loading, though implementations may face accuracy issues when mixing these strategies.

### [Interactive Render Quality Estimation & Cancellation](8895-interactive-render-quality-estimation-cancellation.md)
This method prevents infinite rendering loops and maintains interactivity in large-scale volume rendering by detecting when a frame cannot be completed within a time budget and immediately falling back to a lower-quality estimate. It sets apart from sibling methods like Early Termination and Probabilistic Path Control by explicitly managing the *rendering state machine* across frames rather than just terminating a single ray, ensuring the viewer sees *something* plausible rather than a frozen or black screen. It works in conjunction with Visibility Culling to reduce the workload before the cancellation check is even triggered.

### [Visible Zone Algorithm](9086-visible-zone-algorithm.md)
The Visible Zone Algorithm is a conservative visibility computation method designed for incremental updates in large-scale volume and geometry rendering. It solves the problem of maintaining correct occlusion relationships in interactive rendering by identifying "visible zones" that guarantee no pixel is incorrectly occluded. It sets itself apart from approximate siblings like the Approximate Visibility Algorithm by providing strict correctness guarantees, ensuring less than one pixel of screen-space deviation, at the cost of significant precomputation time.

### Screen Regions
An optimization dividing image space into equal-sized squares of n x n pixels to reduce overhead from per-pixel visibility testing. If a screen region is full, rasterization skips to the next boundary. Screen regions save another 3 percent of overall rendering time on average [S2, unofficial].

### Visibility Culling via Scalar Range Lookup Table
A GPU-based visibility test using a 2*D* Boolean lookup table to cull cells that do not intersect the transfer function's opacity range, reducing load on the rendering stage. Geometry shader computes the minimum and maximum of the 8 cell corner values. These two values are used as table lookup indices to fetch a visibility value. If the cell is deemed visible, the geometry shader instantiates its 6 faces. Visibility Lookup Table: A 2*D* table where D is the number of scalar value bins. For each pair of values s1 and s2, the table holds 1 if there exists at least an opacity which is non-zero for scalar values in [s1, s2], otherwise 0. On a PC with two Xeon E5345 processors, 4GB memory, GeForce 8800 GTX (768MB), using the 'meteorite' dataset (2,377,878 cells), removing the look-up table drops performance to <1 fps (specifically 0.76 fps). Adding the look-up table increases framerate to ~3.48 fps. Adding the hybrid CPU-GPU mesh traversal mechanism to the look-up table optimization increases the framerate to 3.62 fps [S3].

## Operating characteristics
- **Time Complexity**: Linear with respect to the number of rays and the average number of bricks intersected per ray. Implicit culling reduces the effective number of bricks processed [S1, unofficial].
- **Memory Complexity**: Depends on the size of the shared cache texture (brick pool). If the working set exceeds the cache, performance degrades due to paging or multi-pass rendering [S1, unofficial].
- **Measured Performance**: Specific performance metrics are not provided for the general Visibility Culling method in the sources. However, related optimizations like the Scalar Range Lookup Table show significant improvements (from 0.76 fps to 3.48 fps) when culling is applied [S3].
- **Strengths**: Eliminates the need for explicit culling passes; integrates seamlessly with modern GPU architectures; reduces memory bandwidth by only fetching necessary bricks [S1, unofficial].
- **Limitations**: Requires efficient data structures for brick management; performance can degrade if the working set is too large for the cache [S1, unofficial].
- **When to Use**: Suitable for large-scale volume rendering where explicit culling would be too costly or complex to implement [S1, unofficial].
- **When Not to Use**: Not suitable for scenes with extremely small working sets where the overhead of managing the shared cache might outweigh the benefits [G].

## Background
Visibility Culling, also known as Ray-guided volume rendering, was introduced by Crassin et al. in 2009 [S1, unofficial]. It represents a shift from traditional pre-pass culling methods to a more integrated approach that leverages the parallel nature of GPU rendering to determine visibility on the fly. This method has become a standard technique in large-scale scientific visualization, enabling the interactive rendering of massive datasets [S1, unofficial].

## Sources
[S1] GPU-Based Large-Scale Scientific Visualization, https://johanna-b.github.io/LargeSciVis2018/slides/LargeScaleVisualization_Architectures.pdf, fetched 2026-10-29 (unofficial)
[S2] (PDF) Cell-Based First-Hit Ray Casting - ResearchGate, https://www.researchgate.net/publication/2521319_Cell-Based_First-Hit_Ray_Casting, fetched 2026-10-27 (unofficial)
[S3] High-Quality, Semi-Analytical Volume Rendering for AMR ..., https://people.freedesktop.org/~marcheu/vis09-amr.pdf, fetched 2026-10-26
