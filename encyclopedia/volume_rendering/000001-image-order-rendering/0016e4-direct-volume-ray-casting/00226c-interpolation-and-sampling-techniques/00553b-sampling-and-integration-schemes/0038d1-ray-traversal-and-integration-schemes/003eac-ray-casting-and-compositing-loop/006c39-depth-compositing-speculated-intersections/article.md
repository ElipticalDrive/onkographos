# Depth Compositing Speculated Intersections

Also called: Closest Hit Selection, Speculation Depth Compositing

## Overview
This method performs depth compositing on a set of potential intersections found for each ray during speculative ray traversal. It iterates through $N_{Spec}$ candidates to select the closest valid hit and writes the result to the framebuffer, marking rays that exit the volume without a hit as terminated. It differs from non-speculative compositing by handling multiple block IDs per ray ID, requiring an offset buffer to resolve unique ray-block pairs [S1].

## How it works
**Inputs:**
- $N_{Spec}$: Speculation count per ray.
- $RID$: Buffer of ray IDs, where each active ray ID appears $N_{Spec}$ times.
- $O_{Spec}$: Offset buffer providing the index into the intersection data for each ray-block pair.
- Intersection data: Depth values and other attributes for the $N_{Spec}$ blocks traversed by each ray.
- $w, h$: Image width and height.
- $N_{Act}$: Number of currently active rays.

**Outputs:**
- Framebuffer: Final color/depth for each pixel.
- Termination status: Marks rays that did not hit a surface.

**Data Structures:**
- $RID$: Array of size $N_{Act} \times N_{Spec}$. Contains ray identifiers. Not unique per entry; duplicates indicate multiple speculation slots for the same ray [S1].
- $O_{Spec}$: Array of size $N_{Act} \times N_{Spec}$. Stores offsets into the intersection results buffer, allowing the kernel to access the correct depth/value for each specific ray-block intersection [S1].

**Parameters:**
- $N_{Spec} = \lfloor \frac{w \times h}{N_{Act}} \rfloor$ [S1].
- Maximum $N_{Spec} = 64$ [S1].

**Steps:**
1. For each active ray $r$ in the active set:
2.   Initialize $closest\_hit = \infty$ and $hit\_found = false$.
3.   For $i$ from 0 to $N_{Spec} - 1$:
4.     Retrieve the ray ID $rid = RID[r \times N_{Spec} + i]$.
5.     Retrieve the offset $offset = O_{Spec}[r \times N_{Spec} + i]$.
6.     Retrieve the depth $d$ and attributes from the intersection data at $offset$.
7.     If $d < closest\_hit$:
8.       $closest\_hit = d$
9.       $best\_offset = offset$
10.      $hit\_found = true$
11.   If $hit\_found$:
12.     Write the attributes from $best\_offset$ to the framebuffer at the pixel corresponding to ray $r$.
13.   Else:
14.     Mark ray $r$ as terminated.

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def depth_compositing_speculated_intersections(RID, O_Spec, intersection_data, N_Spec, w, h, N_Act):
    """
    Selects the closest intersection for each active ray from N_Spec candidates.
    """
    # Initialize framebuffer and termination status
    framebuffer = [None] * (w * h)
    terminated = [False] * N_Act

    for r in range(N_Act):
        closest_depth = float('inf')
        best_result = None
        hit_found = False

        # Iterate through all speculated intersections for this ray
        for i in range(N_Spec):
            # Get the specific intersection data using the offset buffer
            # Note: RID[r * N_Spec + i] identifies the ray, but O_Spec provides the index
            # into the flat intersection results array for this specific block hit
            offset = O_Spec[r * N_Spec + i]
            
            # Retrieve depth and attributes from the pre-computed intersection data
            depth = intersection_data[offset].depth
            attributes = intersection_data[offset].attributes

            # Select the closest hit
            if depth < closest_depth:
                closest_depth = depth
                best_result = attributes
                hit_found = True

        if hit_found:
            # Write the closest hit to the framebuffer
            # Assuming ray r maps to a specific pixel in the image
            pixel_idx = r # Simplified mapping; in practice, ray r corresponds to a specific (x,y)
            framebuffer[pixel_idx] = best_result
        else:
            # Ray exited volume without a hit
            terminated[r] = True

    return framebuffer, terminated
```

**Edge Cases and Pitfalls:**
- If $N_{Act}$ is zero, $N_{Spec}$ is undefined; handle as empty pass [S1].
- Precision issues may arise if depth values are very close; standard float comparison is used [S1].
- Without 64-bit atomics, depth sorting must be done in this compositing step rather than during traversal [S1].

## Subtopics

## Operating characteristics
- **Time Complexity:** $O(N_{Act} \times N_{Spec})$ per pass. Since $N_{Spec} \approx \frac{w \times h}{N_{Act}}$, total work is $O(w \times h)$ per pass, independent of $N_{Act}$ [S1].
- **Memory Complexity:** Requires buffers for $RID$ and $O_{Spec}$ of size $N_{Act} \times N_{Spec} \approx w \times h$ [S1].
- **Performance:** Evaluated on datasets from $256^3$ to $2048 \times 2048 \times 1920$ voxels [S1].
- **Strengths:** Allows efficient use of GPU resources by reusing terminated ray slots for speculation [S1].
- **Limitations:** Requires multiple passes if $N_{Spec}$ is small; limited to 64 speculation slots [S1].

## Background
Originated in speculative progressive raycasting for memory-constrained isosurface visualization of massive volumes [S1]. The method addresses the non-uniqueness of ray IDs in speculative traversal by introducing offset buffers [S1].

## Sources
[S1] Speculative Progressive Raycasting for Memory Constrained Isosurface Visualization of Massive Volumes, https://doi.org/10.48550/arxiv.2309.10212, fetched 2026-10-26
