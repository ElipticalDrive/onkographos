# Hardware-Assisted Volume Streaming

Also called: HAVS (Hardware-Assisted Volume Streaming), HAVS2, HAVS6

## Overview
Hardware-Assisted Volume Streaming (HAVS) is an irregular volume rendering method that streams volumetric data to perform rendering, offering more accurate visibility sorting than centroid-based approaches like HAPT [S1, unofficial]. It distinguishes itself from methods requiring full volume upload to GPU memory (e.g., PTINT, HARC) by streaming chunks, thereby avoiding memory limitations and the performance penalties associated with repeated data transfers [S1, unofficial].

## How it works
**Inputs and Outputs:**
*   **Input:** Volumetric data (e.g., tetrahedral meshes), camera parameters (origin $\mathbf{o}$, direction $\mathbf{d}$), transfer functions, step size $s$, and maximum ray distance $D_{max}$ [S1, unofficial][G].
*   **Output:** Rendered image [S1, unofficial].

**Data Structures:**
Not given in the sources.

**Steps:**
1.  Partition the volume into chunks suitable for streaming to the rendering hardware [S1, unofficial].
2.  Stream each chunk to the GPU [S1, unofficial].
3.  Perform visibility sorting on the streamed primitives, which is more accurate than the centroid sort methods used in HAPT (HAPTQ, HAPTB, HAPTS) [S1, unofficial].
4.  Render the sorted primitives using hardware-assisted fragment processing and front-to-back compositing [S1, unofficial].

**Formulas:**
Front-to-back volume compositing for each ray step [G]:
$$C_{out} = C_{in} + c \cdot (1 - \alpha_{in})$$
$$\alpha_{out} = \alpha_{in} + \alpha \cdot (1 - \alpha_{in})$$
where $C_{in}, \alpha_{in}$ are the accumulated color and alpha, and $c, \alpha$ are the sampled color and alpha at the current position.

**Parameters:**
Step size $s$ (typical values not given in the sources) [S1, unofficial].

**Edge Cases and Pitfalls:**
*   **Memory Limits:** Unlike PTINT or HARC, HAVS does not require uploading the entire volume to GPU memory, avoiding the need for multiple upload/download cycles for animation frames [S1, unofficial].
*   **Sorting Accuracy:** While more accurate than centroid sorts, the specific implementation details of the sorting algorithm (e.g., exact k-buffer size or sorting logic) are not provided in the source text for the general HAVS method [S1, unofficial].
*   **Direction Normalization:** Unnormalized direction vectors will cause incorrect step progression; the direction must be normalized before ray marching [G].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def render_havs(origin, direction, step_size, max_distance, volume_sample_fn, transfer_fn):
    # Normalize direction to ensure consistent step progression [G]
    length = (direction[0]**2 + direction[1]**2 + direction[2]**2)**0.5
    if length > 0:
        direction = [d / length for d in direction]
    
    # Initialize accumulators (fixes the zero-output bug from uninitialized state)
    accum_color = [0.0, 0.0, 0.0]
    accum_alpha = 0.0
    t = 0.0
    
    # Stream and process chunks (exact chunking and sorting pipeline not detailed in source [S1, unofficial])
    # Process a single ray through the current chunk:
    while accum_alpha < 1.0 and t < max_distance:
        # Compute current sample position
        pos = [origin[i] + t * direction[i] for i in range(3)]
        
        # Sample volume and apply transfer function
        sample_color, sample_alpha = volume_sample_fn(pos)
        c, a = transfer_fn(sample_color, sample_alpha)
        
        # Front-to-back compositing
        accum_color = [accum_color[i] + c[i] * (1.0 - accum_alpha) for i in range(3)]
        accum_alpha += a * (1.0 - accum_alpha)
        
        t += step_size
        
    return accum_color
```

## Subtopics

### [Hardware Assisted Visibility Sorting (HAVS)](2918-hardware-assisted-visibility-sorting-havs.md)
A hybrid volume rendering algorithm for unstructured grids that combines object-space sorting on the CPU with image-space fragment sorting on the GPU. It uses a fixed-depth "k-buffer" to incrementally sort fragments and composite colors, solving the visibility ordering problem for arbitrary non-convex meshes without topological connectivity.

## Operating characteristics
**Time and Memory Complexity:**
Not given in the sources.

**Measured Performance:**
*   **Torso Dataset (HAVS2):** 3.7 FPS, 4.01 M Tet/s [S1, unofficial].
*   **Fighter Dataset (HAVS2):** 2.9 FPS, 4.12 M Tet/s [S1, unofficial].
*   **HAVS6 Performance:** 3.3 FPS, 3.60 M Tet/s (torso); 2.7 FPS, 3.89 M Tet/s (fighter) [S1, unofficial].
*   **Comparison:** HAVS6 offers a gain of approximately 50% in frame rates compared to HAPTQ and does not require multiple rendering passes [S1, unofficial].

**Scaling:**
Not given in the sources.

**Strengths and Limitations:**
*   **Strengths:** More accurate visibility sorting than HAPT centroid sorts; avoids memory limitations by streaming; flexible pipeline [S1, unofficial].
*   **Limitations:** Performance is lower than some other methods (e.g., HARCn, HARCp) in raw FPS comparisons on the tested datasets, though those methods require significantly more memory (144 bytes/tet and 96 bytes/tet respectively) [S1, unofficial].

**When to Use:**
Use when the volume is too large for GPU memory and accurate visibility sorting is required, particularly for irregular volumes like tetrahedral meshes [S1, unofficial].

## Background
**Origin:**
HAVS is described as one of the most popular irregular volume renderers [S1, unofficial]. It is part of the HAPT (Hardware-Assisted Projected Tetrahedra) family of algorithms [S1, unofficial].

**History:**
The method was discussed in the context of comparing various irregular volume rendering algorithms, including HAPT variants (HAPTQ, HAPTB, HAPTS, HAPTM), HARC, PTINT, GATOR, and HAVS variants (HAVS2, HAVS6) [S1, unofficial].

**Usage:**
Used for rendering irregular volumetric data, such as tetrahedral meshes, where memory constraints prevent storing the entire volume on the GPU [S1, unofficial].

## Sources
[S1] (PDF) Hardware-Assisted Projected Tetrahedra - Academia.edu, https://www.academia.edu/16536671/Hardware_Assisted_Projected_Tetrahedra, fetched 2026-10-28 (unofficial)
