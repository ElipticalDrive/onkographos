# A-buffer linked-list creation and ray processing

Also called: A-buffer linked-list creation, Ray processing with A-buffer linked-list

## Overview
This method constructs a dynamic acceleration structure (A-buffer) on the GPU to efficiently identify which segment primitives overlap a given ray, avoiding the evaluation of empty space in expensive field computations [S1]. It differs from brute-force ray marching by using a geometry shader to generate quads covering primitive supports and a fragment shader to compute ray-support intersections, storing entry/exit points in a linked list [S1]. Unlike [Bru19], which requires explicit cutoff of primitive influences, this approach leverages compact support kernels (specifically clipped sphere-cones for Cauchy kernels) to naturally limit the interaction domain [S1].

## How it works
**Inputs:**
*   Skeleton geometry defined as a set of segment primitives.
*   Kernel parameters (e.g., scale $\sigma$, clipping radius).
*   View ray $(i, j)$ for each pixel.
*   Optional occluding depth buffer for output sensitivity.

**Outputs:**
*   Isosurface intersection point or null if no intersection is found within bounds.

**Data Structures:**
1.  **A-Buffer Linked List:** Stores fragments per pixel. Each fragment contains:
    *   Entry depth ($z_{in}$) along the ray.
    *   Exit depth ($z_{out}$) along the ray.
    *   Primitive ID.
    *   The list is sorted by entry depth $z_{in}$ [S1].
2.  **Processing Heaps (per ray):**
    *   `primitives`: A min-heap (or sorted list) of active primitives, keyed by their exit depth ($z_{out}$). Stores tuples $(z_{out}, \text{segId})$.
    *   `argmins`: A min-heap storing the minimum field value (or derivative metric) contributed by each active primitive at the current ray position. Stores tuples $(\text{argmin\_val}, \text{segId})$.

**Steps:**

1.  **Initialization:**
    *   Initialize `primitives` and `argmins` as empty heaps [S1].
    *   Initialize `frags` as the linked list of fragments for pixel $(i, j)$, obtained via `InitFragList(i, j)` [S1].
    *   Set `targmin` $\leftarrow \infty$ [S1].
    *   Set `tentry` $\leftarrow$ `frags.currZIn()` [S1].

2.  **Occlusion Culling (Optional but recommended):**
    *   Render an occluding depth buffer in a first pass to determine the maximum depth $z_{max}$ guaranteed to be inside the volume [S1].
    *   Ignore any primitive whose support entry $z_{in}$ is beyond $z_{max}$ [S1].
    *   Compute scaling factors for sphere-cone radii based on the worst-case skeleton configuration and kernel scale to ensure the implicit surface encloses modified primitives [S1]. For compact polynomial kernels with $\sigma \in [1.5, 2]$, scaling factors range from $\approx 0.62$ to $\approx 0.35$ [S1].

3.  **Ray Processing Loop:**
    While `!frags.empty()` OR `!primitives.empty()`:
    
    a.  **Prepare Current Interval:**
        *   Store current state: $t_b, h_b \leftarrow t_e, h_e$ [S1].
        *   Get potential exit from active primitives: $t_{exit} \leftarrow$ `primitives.back().val` (the largest $z_{out}$ in the heap, or $\infty$ if empty) [S1].
        
    b.  **Determine Next Event:**
        *   If `primitives.empty()` OR `targmin < t_exit` OR `tentry < t_exit`:
            *   If `primitives.empty()`:
                *   Set interval start $t_b, h_b \leftarrow t_{entry}, (g(0), \dots)$ [S1].
                *   *Note:* $g(0)$ represents the field value where the field is null [S1].
            *   **Refill Active Primitives:**
                *   While `targmin > tentry` AND `!frags.empty()`:
                    *   Get primitive ID from current fragment: `segId` $\leftarrow$ `frags.currSegId()` [S1].
                    *   Compute the argument minimizing the field contribution for this primitive: $t_{prim} \leftarrow \text{Argmin}(\sigma, \text{segId})$ [S1].
                    *   Get fragment exit depth: $t_{out} \leftarrow$ `frags.currZOut()` [S1].
                    *   Insert into active primitives heap: `primitives.insert({t_out, segId})` [S1].
                    *   Insert into argmins heap: `argmins.insert(t_prim)` [S1].
                    *   Update target minimum: `targmin` $\leftarrow$ `argmins.front()` [S1].
                    *   Advance fragment: `frags.next()` [S1].
                    *   Update next entry: $t_{entry} \leftarrow$ (`!frags.empty()`) ? `frags.currZIn()` : $\infty$ [S1].
            *   Set interval start: $t_e, h_e \leftarrow t_{argmin}, \text{HermiteData}(t_{argmin})$ [S1].
        *   Else:
            *   Set interval start: $t_e, h_e \leftarrow t_{exit}, (g(0), \dots)$ [S1].
            
    c.  **Process Interval:**
        *   If $t_b < t_e$:
            *   Evaluate field and derivative over $[t_b, t_e]$: $f, t \leftarrow \text{processInterval}(t_b, h_b, t_e, h_e)$ [S1].
            *   If intersection found ($f$ is true):
                *   Generate isosurface at $t$: `GenerateIsoSurface(t)` [S1].
                *   Return [S1].
                
    d.  **Cleanup:**
        *   Remove primitives that have exited the current interval: `primitives.popElements([](x){x \le t_e})` [S1].
        *   Remove argmins that have exited: `argmins.popElements([](x){x \le t_e})` [S1].
        *   Update target minimum: `targmin` $\leftarrow$ (`!argmins.empty()`) ? `argmins.front()` : $\infty$ [S1].

**Edge Cases and Pitfalls:**
*   **Numerical Instability:** The normalized field derivative is discontinuous when the field switches from null to non-null [S1]. Apply one step of sphere tracing to avoid these areas. This requires no field evaluation as the field is null at the support limit [S1].
*   **Bent Skeletons:** Due to the integral formulation, the minimal radius around primitives may not match the prescribed radius [S1]. Use skeleton end-point correctors and assume bounded radius variations (neither sphere at a primitive endpoint completely encompasses the other) [S1]. Do not render occluders for primitives corresponding to end-point correctors [S1].
*   **Blending Artifacts:** Sphere-tracing and Sherstyuk's fast ray tracing can present visual artifacts in the presence of large amounts of blending (e.g., many segments connecting at one vertex) [S1]. Use under-relaxation to properly find the surface [S1].
*   **Subdivision Strategy:** Use four subdivisions per primitive for the coarsest subdivision allowing visually acceptable results on primitives with constant radius [S1]. Limit sphere-tracing steps per subinterval (typically 256) [S1].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def GenerateRay(i, j, frags, kernel_scale):
    """
    Process a single ray using the A-buffer linked list.
    
    Args:
        i, j: Pixel coordinates.
        frags: Linked list of fragments for pixel (i, j), sorted by z_in.
        kernel_scale: Kernel scale parameter sigma.
        
    Returns:
        Intersection point or None.
    """
    # Heaps to store active primitives and their minimum field contributions
    # primitives: min-heap of (z_out, seg_id)
    # argmins: min-heap of (min_field_val, seg_id)
    primitives = empty_min_heap()
    argmins = empty_min_heap()
    
    # Initialize fragment iterator
    frags = InitFragList(i, j)
    
    # Current state variables
    targmin = infinity
    tentry = frags.currZIn()
    te = 0.0
    he = 0.0
    
    while not (frags.empty() and primitives.empty()):
        # Store previous interval end for processing
        tb, hb = te, he
        
        # Determine the exit point of the current batch of active primitives
        if primitives.empty():
            texit = infinity
        else:
            # back() returns the element with the largest key in a min-heap? 
            # Note: The source says "primitives.back().val". 
            # In a min-heap keyed by z_out, back() is not standard. 
            # Assuming a sorted list or max-heap for exit times, or that 
            # primitives stores intervals and we need the max z_out.
            # Based on "primitives.popElements([](x){x <= te})", it seems 
            # we remove elements <= te. This implies we need to access 
            # the element with the largest z_out to know when the *current* 
            # set of primitives ends. Let's assume primitives is a structure 
            # allowing access to the max z_out.
            texit = primitives.max_z_out()
            
        # Check if we need to pull more fragments from the A-buffer
        if primitives.empty() or targmin < texit or tentry < texit:
            if primitives.empty():
                # No active primitives, field is null
                tb, hb = tentry, (0.0, 0.0, 0.0, 0.0) # g(0)
            
            # Refill active primitives from the fragment list
            while targmin > tentry and not frags.empty():
                seg_id = frags.currSegId()
                
                # Compute the argument (t) that minimizes the field for this primitive
                tprim = Argmin(kernel_scale, seg_id)
                
                # Get the exit depth of the current fragment
                tout = frags.currZOut()
                
                # Add to active sets
                primitives.insert(tout, seg_id)
                argmins.insert(tprim, seg_id)
                
                # Update the global minimum
                targmin = argmins.front()
                
                # Advance fragment
                frags.next()
                
                # Update next entry point
                if not frags.empty():
                    tentry = frags.currZIn()
                else:
                    tentry = infinity
            
            # Set the start of the next interval to the argmin
            te, he = targmin, HermiteData(targmin)
        else:
            # The next event is the exit of the current active primitives
            te, he = texit, (0.0, 0.0, 0.0, 0.0) # g(0)
            
        # Process the interval [tb, te]
        if tb < te:
            # Evaluate field and derivative over the interval
            # Returns (found_intersection, t_intersection)
            f, t = processInterval(tb, hb, te, he)
            
            if f:
                GenerateIsoSurface(t)
                return t
                
        # Remove primitives that have exited the current interval
        # Note: The source uses a lambda to pop elements <= te.
        # This implies we need to efficiently remove all primitives with z_out <= te.
        primitives.popElements(lambda x: x.z_out <= te)
        argmins.popElements(lambda x: x.z_out <= te) # Assuming argmins is keyed/linked similarly
        
        # Update targmin
        if not argmins.empty():
            targmin = argmins.front()
        else:
            targmin = infinity
            
    return None
```

## Subtopics

### [GPU A-Buffer Construction for Ray-Primitive Intersection](4000-gpu-a-buffer-construction-for-ray-primitive-intersection.md)
This method constructs a dynamic acceleration structure (A-buffer) on the GPU to efficiently identify which segment primitives overlap a given ray, avoiding the evaluation of empty space in expensive field computations. It differs from brute-force ray marching by using a geometry shader to generate quads covering primitive supports and a fragment shader to compute ray-support intersections, storing entry/exit points in a linked list. Unlike [Bru19], which requires explicit cutoff of primitive influences, this approach leverages compact support kernels (specifically clipped sphere-cones for Cauchy kernels) to naturally limit the interaction domain.

### [A-Buffer Visibility Pass](16279-a-buffer-visibility-pass.md)
The A-Buffer Visibility Pass is a GPU fragment-stage algorithm that generates an approximate visibility function $\tilde{v}(x_e, x)$ for each view ray by storing depth and visibility control points in an A-Buffer. It solves the problem of efficiently representing volumetric occlusion along semi-transparent view rays to enable optimized illumination calculations, such as determining entry points for light rays or approximating multiple scattering. Unlike sibling methods that may focus on primitive intersection or general buffer construction, this method specifically targets the generation of a sparse, error-thresholded visibility function using a fixed maximum number of layers, similar in concept to deep shadow maps but applied to view-space volumetric visibility.

## Operating characteristics
**Time and Memory Complexity:**
*   The method limits the number of segment primitives stored at once by combining subdivision strategies, trading off memory usage per thread for fewer compulsory field evaluations [S1].
*   Rendering time reduction compared to sphere tracing ranges from -46.2% to -81.3% on nVidia RTX at 1536x1536 resolution for Cauchy kernels [S1].
*   For compact polynomial kernels at 2048x2048 resolution, rendering time reductions range from -34.2% to -48.7% on Intel UHD Graphics 630, -51.5% to -78.0% on nVidia Quadro P1000, and -35.3% to -72.1% on nVidia GeForce 2080 RTX compared to the most competitive alternative method [S1].

**Measured Performance:**
*   On nVidia RTX at 1536x1536 resolution with Cauchy kernel ($\sigma=0.3, \epsilon_{clipping}=0.005$), timings range from 7 to 54 milliseconds (excluding specific figures) [S1].
*   Stress test: Rendered a foam structure with 526k segments at 15 frames per second using the compact polynomial kernel on nVidia RTX [S1].

**Strengths and Limitations:**
*   **Strengths:** Avoids evaluation of empty space; output sensitivity via occluding depth buffer reduces primitive count; compact support kernels eliminate need for explicit cutoff [S1].
*   **Limitations:** Numerical instability at field boundaries requires sphere tracing steps; blending artifacts may require under-relaxation; larger kernel footprints (e.g., Cauchy) increase computational time and A-buffer overflow risk [S1].

**When to Use:**
*   Use for real-time rendering of skeleton-based implicit surfaces where empty space evaluation is expensive.
*   Use when compact support kernels are applicable.
*   Do not use if high blending density causes severe artifacts without significant performance penalty from under-relaxation [S1].

## Background
The method is implemented in C++ using OpenGL, with shaders in GLSL [S1]. It was tested on Intel UHD Graphics 630, nVidia Quadro P1000, and nVidia GeForce 2080 RTX [S1]. Test cases included artistic shapes, truss structures, branching structures, procedural foam structures, and random skeletons [S1]. The approach builds on [Atomic Fragment List Insertion](2465-atomic-fragment-list-insertion.md) and is compared to sphere-tracing [Bru19], Sherstyuk fast ray-tracing [She99], and segment-tracing [GGPP20] [S1]. Blending can be sharper than summation blending using Ricci's blending [Ric73] [S1].

## Sources
[S1] Fast Ray Tracing of Scale‐Invariant Integral Surfaces, https://doi.org/10.1111/cgf.14208, fetched 2026-10-26
