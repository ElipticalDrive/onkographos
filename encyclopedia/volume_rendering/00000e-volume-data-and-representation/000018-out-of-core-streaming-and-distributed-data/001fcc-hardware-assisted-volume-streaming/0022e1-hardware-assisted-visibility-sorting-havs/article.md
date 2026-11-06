# Hardware Assisted Visibility Sorting (HAVS)

Also called: Hardware-Assisted Volume Rendering System, Hardware-assisted visibility ordering

## Overview
Hardware Assisted Visibility Sorting (HAVS) is a hybrid volume rendering algorithm for unstructured grids that combines object-space sorting on the CPU with image-space fragment sorting on the GPU [S2][S3, S4, unofficial]. It solves the visibility problem for arbitrary non-convex meshes without requiring topological connectivity or preprocessing [S3, S4, S5, unofficial]. Unlike sorting-free methods like Projected Tetrahedra (SPT) which avoid artifacts but may be slower, or pure ray-casting which is memory-bound by texture units, HAVS uses a k-buffer to incrementally sort fragments, offering a compromise between implementation complexity, robustness, and image quality [S1][S3, S4, S5, unofficial]. It builds on [Hardware-Assisted Volume Streaming](2119-hardware-assisted-volume-streaming.md) by providing a more accurate visibility sort than centroid-based methods [Parent].

## How it works
HAVS operates in two phases: object-space partial sorting on the CPU and image-space complete sorting/compositing on the GPU using a k-buffer [S3, S4, unofficial].

### Inputs and Outputs
*   **Input:** Unstructured mesh (e.g., tetrahedral cells), scalar field values at vertices, transfer function (pre-integrated 3D table), camera parameters (ModelView, Projection matrices).
*   **Output:** Final color and opacity framebuffer.

### Data Structures
*   **Face List:** Array of faces (triangles) extracted from tetrahedral cells. Each face has a centroid and associated scalar values at vertices. Vertices are marked as internal or boundary to handle mesh edges correctly [S3, S4, unofficial].
*   **k-buffer:** A fixed-size buffer of length $k$ per pixel. Stores up to $k$ fragments. Each fragment entry contains:
    *   Scalar value ($v$): The data value at the fragment's position.
    *   Distance ($d$): The distance from the eye/camera to the fragment.
    *   Layout: Stored in 2D textures. For $k=6$, three textures store pairs of entries (entry 1&2, 3&4, 5&6) [S3, unofficial].
*   **Pre-integrated Transfer Function:** A 3D texture of size $128^3$ (8-bit or 16-bit RGBA) mapping front scalar ($v_1$), back scalar ($v_2$), and distance ($d_2 - d_1$) to accumulated color and opacity [S3, unofficial].
*   **Framebuffer:** A floating-point 2D texture (pbuffer) storing accumulated color and opacity [S3, unofficial].

### Parameters
*   $k$: Buffer size. Typical values are 2 or 6 [S3, unofficial]. Larger $k$ increases accuracy but reduces performance [S2].
*   Sorting Network Depth: Determines the number of comparisons in the GPU fragment shader. Can be adjusted to trade off CPU sorting accuracy for GPU workload [S3, unofficial].
*   Opacity Threshold: Early termination threshold (e.g., 0.99) [S3, unofficial].

### Steps

1.  **Object-Space Sorting (CPU):**
    *   Extract faces from the unstructured mesh.
    *   Compute the centroid of each face.
    *   Sort the faces based on centroid distance from the camera.
    *   Algorithm: Use LSD radix sort and counting sort. Convert floating-point coordinates to 32-bit unsigned integers using the bit-manipulation formula to preserve order:
        $$ f = f \times (-(f>>31) | 0x80000000) $$
        where $f$ is the coordinate, `>>` is bitwise right shift, and `|` is bitwise OR [S2].
    *   Complexity: $O(n + k)$, where $n$ is the number of faces [S2].

2.  **Rasterization and Fragment Generation (GPU):**
    *   For each sorted face, rasterize its triangles.
    *   **Vertex Shader:**
        *   Transform vertex to clip coordinates using MVP matrix.
        *   Transform vertex to eye coordinates using ModelView matrix.
        *   Pass scalar data value (interpolated) to fragment shader.
    *   **Fragment Shader:**
        *   Compute texture coordinates for framebuffer lookup from window position (to avoid perspective interpolation errors).
        *   Lookup current accumulated color/opacity from framebuffer.
        *   **Early Termination:** If accumulated opacity $\ge 0.99$, kill the fragment [S3, unofficial].
        *   Load current fragment's scalar ($v_{new}$) and distance ($d_{new}$).
        *   Load existing $k$ fragments from the k-buffer textures.
        *   **k-buffer Sort (Sorting Network):**
            *   Find the nearest fragment ($r_0$) and the second nearest fragment ($r_7$) among the $k+1$ candidates (current + $k$ buffer entries) using a fixed-depth comparison network [S3, unofficial].
            *   The network uses `CMP` operations to select minimums and discard maximums iteratively.
        *   **Transfer Function Lookup:**
            *   Use $r_0.x$ (front scalar), $r_7.x$ (back scalar), and $r_7.y - r_0.y$ (distance) as coordinates for the 3D pre-integrated transfer function texture.
            *   Lookup accumulated color ($C$) and opacity ($\alpha$).
            *   If $r_0.x < 0$, nullify the result (invalid scalar) [S3, unofficial].
        *   **Compositing:**
            *   Composite the new color/opacity with the framebuffer color using front-to-back blending:
                $$ C_{out} = C \cdot (1 - \alpha_{fb}) + C_{fb} $$
                $$ \alpha_{out} = \alpha + \alpha_{fb} \cdot (1 - \alpha) $$
        *   **Update k-buffer:**
            *   Discard the nearest fragment ($r_0$).
            *   Write the remaining $k$ fragments back to the k-buffer textures [S3, unofficial].

### Pseudocode

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
# CPU Phase
def cpu_sort(mesh_faces, camera_pos):
    """Sort faces by centroid distance to camera."""
    for face in mesh_faces:
        face.centroid = compute_centroid(face.vertices)
        face.dist = distance(face.centroid, camera_pos)
    # LSD Radix Sort on face.dist
    sorted_faces = radix_sort(mesh_faces, key='dist')
    return sorted_faces

# GPU Phase (Fragment Shader Logic)
def fragment_shader(current_fragment, k_buffer, framebuffer, tf_table):
    """
    current_fragment: {scalar: v, distance: d}
    k_buffer: List of k fragments [{scalar: v, distance: d}, ...]
    framebuffer: {color: C, opacity: alpha}
    """
    
    # 1. Early Termination
    if framebuffer.opacity >= 0.99:
        discard()

    # 2. Load k-buffer entries
    # k_buffer is stored in textures; load all k entries
    entries = load_k_buffer(k_buffer)
    
    # 3. Combine current fragment with buffer
    candidates = [current_fragment] + entries
    
    # 4. Find nearest (r0) and second nearest (r7) using sorting network
    # This is a fixed-depth network of comparisons
    r0, r7 = find_two_nearest(candidates)
    
    # 5. Transfer Function Lookup
    # r0 is front, r7 is back
    front_scalar = r0.scalar
    back_scalar = r7.scalar
    segment_dist = r7.distance - r0.distance
    
    # Normalize distance for texture lookup if needed
    normalized_dist = scale_and_bias(segment_dist)
    
    # Lookup pre-integrated color and opacity
    color, alpha = tf_table.lookup(front_scalar, back_scalar, normalized_dist)
    
    # Nullify if scalar is invalid
    if front_scalar < 0:
        color = (0, 0, 0, 0)
        alpha = 0

    # 6. Composite with framebuffer
    # Front-to-back blending
    one_minus_fb_alpha = 1.0 - framebuffer.opacity
    out_color = color * one_minus_fb_alpha + framebuffer.color
    out_alpha = alpha + framebuffer.opacity * one_minus_fb_alpha
    
    # 7. Update k-buffer
    # Discard r0 (nearest), keep the rest
    remaining_entries = remove_fragment(candidates, r0)
    
    # Write remaining entries back to k-buffer textures
    write_k_buffer(remaining_entries)
    
    return out_color, out_alpha

def find_two_nearest(candidates):
    """
    Find the fragment with min distance (r0) and the next min (r7)
    from the list of candidates using a sorting network.
    """
    # Example for k=6 (7 candidates total)
    # This logic is unrolled in the actual shader using CMP instructions
    # r0 = min(candidates)
    # r7 = min(candidates - {r0})
    
    # Simplified logic for description:
    r0 = min(candidates, key=lambda x: x.distance)
    others = [c for c in candidates if c is not r0]
    r7 = min(others, key=lambda x: x.distance)
    
    return r0, r7
```

### Edge Cases and Pitfalls
*   **Out-of-Order Fragments:** If fragments are highly out of order (e.g., due to poor CPU sorting or large $k$ buffer overflow), the k-buffer may miss the true nearest fragments, leading to incorrect compositing and visible artifacts [S1, S2].
*   **Hardware Support:** Requires advanced read/write framebuffer access (e.g., `ATI_draw_buffers` or similar extensions), which was historically unstable on some hardware but works on ATI/NVIDIA [S4, S5, unofficial].
*   **Perspective Interpolation:** Texture coordinates for framebuffer lookup must be computed from window position in the fragment shader, not interpolated, to avoid perspective errors [S3, unofficial].
*   **Invalid Scalars:** Scalar values $< 0$ must be handled to prevent invalid transfer function lookups [S3, unofficial].

## Subtopics

### [HAVS with Dynamic LOD](4110-havs-with-dynamic-lod.md)
An algorithm for dynamic Level of Detail (LOD) that builds on [Hardware Assisted Visibility Sorting (HAVS)](2918-hardware-assisted-visibility-sorting-havs.md), separating boundary and internal faces, sorting internal faces by importance, and dynamically adjusting the number of faces drawn based on frame rate. It solves the problem of maintaining interactive frame rates on large tetrahedral datasets by progressively streaming approximations that increase in quality during idle frames. Unlike [Optimized HAVS (Sorted k-buffer)](4735-optimized-havs-sorted-k-buffer.md), which focuses on sorting optimization, this method focuses on adaptive primitive reduction to balance interactivity and visual fidelity.

### [Optimized HAVS (Sorted k-buffer)](4735-optimized-havs-sorted-k-buffer.md)
Optimized HAVS is a refinement of Hardware Assisted Visibility Sorting [Hardware Assisted Visibility Sorting (HAVS)](2918-hardware-assisted-visibility-sorting-havs.md) that maintains the k-buffer in a strictly sorted state (ascending order of fragment distance) to reduce the computational cost of fragment-level visibility sorting. While standard HAVS requires $2k-1$ comparisons to identify the two nearest fragments for compositing, this optimization reduces that requirement to $k$ comparisons per fragment processing step after the buffer is full, thereby improving rendering efficiency for large unstructured grid datasets.

## Operating characteristics
*   **Time Complexity:** Object-space sorting is $O(n + k)$ [S2]. Image-space sorting is $O(1)$ per fragment for fixed $k$, but the constant factor depends on the sorting network depth. Total comparisons per pixel for unsorted k-buffer is $N_1 = \sum_{j=1}^{m \times n} (\sum_{l=2}^k (2l-3) + (k+k-1) \times (s_j - k))$ [S2].
*   **Memory Complexity:** Low memory overhead as it requires no topological information [S3, unofficial]. Memory usage is dominated by the k-buffer (small) and the pre-integrated transfer function ($128^3$) [S3, unofficial].
*   **Performance:** On an ATI Radeon 9800 with $k=2$, HAVS achieved 6.8 fps (1622K tets/sec) for the F117 dataset [S3, unofficial]. With $k=6$, performance dropped to 4.42 fps (1062K tets/sec) for the same dataset [S3, unofficial]. Total performance (CPU+GPU) for F117 was 148ms [S3, unofficial]. With early ray termination, over 6 million cells/sec were achieved [S3, unofficial].
*   **Strengths:** Simple implementation (<200 lines of rendering code) [S3, unofficial], handles arbitrary non-convex meshes, no preprocessing required, interactive transfer function updates [S3, unofficial].
*   **Limitations:** Sorting errors can cause artifacts [S1]. Performance degrades with larger $k$ or highly unsorted fragments [S2]. Requires specific GPU hardware features for read/write framebuffer [S4, unofficial].

## Background
HAVS was introduced by Steven Callahan, Milan Ikits, Joao Comba, and Claudio Silva in 2005 [S3, S4, unofficial]. It was presented as a novel algorithm for volume rendering unstructured data sets, leveraging the concurrent processing power of CPU and GPU. The method was highlighted as the best compromise in implementation complexity, robustness, and image quality among surveyed techniques at the time [S4, S5, unofficial]. It is used in the HAVS system for rendering large unstructured grids and has been extended to isosurface extraction and dynamic LOD [S3, unofficial].

## Sources
[S1] Sorting-free pre-integrated projected tetrahedra, https://doi.org/10.1145/1838544.1838546, fetched 2026-11-01
[S2] Optimization and Implementation of Unstructured Grid Volume Rendering Algorithm, https://doi.org/10.2991/iccasm.2012.373, fetched 2026-10-26
[S3] THE K-BUFFER AND ITS APPLICATIONS TO VOLUME RENDERING, https://ctsilva.github.io/thesis/steve-callahan-ms-thesis.pdf, fetched 2026-10-27 (unofficial)
[S4] PDF A Survey of GPU-Based Volume Rendering of Unstructured Grids, https://www.sci.utah.edu/~stevec/papers/gputet_survey.pdf, fetched 2026-10-27 (unofficial)
[S5] A Survey of GPU-Based Volume Rendering of Unstructured Grids., https://www.researchgate.net/publication/220162223_A_Survey_of_GPU-Based_Volume_Rendering_of_Unstructured_Grids, fetched 2026-10-28 (unofficial)
[S6] Volume rendering for curvilinear and unstructured grids, https://www.researchgate.net/publication/4023713_Volume_rendering_for_curvilinear_and_unstructured_grids, fetched 2026-10-30 (unofficial)
