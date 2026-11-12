# Hardware Assisted Visibility Sorting (HAVS)

## Overview
Hardware Assisted Visibility Sorting (HAVS) is a hybrid volume rendering algorithm for unstructured grids that combines object-space sorting on the CPU with image-space fragment sorting on the GPU using a k-buffer [S3, unofficial][S8]. It solves the problem of rendering large, non-convex unstructured meshes where explicit fragment ordering is required, unlike regular grids which can rely on generation order [S8]. HAVS is distinguished from its siblings by its reliance on a fixed-depth sorting network (k-buffer) to handle visibility, offering a robust compromise between implementation complexity, image quality, and performance, though it may produce artifacts if fragments are highly out of order [S1][S5, unofficial].

## How it works
HAVS operates in two phases: object-space partial sorting on the CPU and image-space exact sorting/compositing on the GPU [S3, unofficial][S8].

### Inputs and Outputs
*   **Input:** Unstructured mesh (e.g., tetrahedral cells), scalar field data, transfer function, camera parameters.
*   **Output:** Rendered image (color and opacity) in the framebuffer.

### Data Structures
*   **k-buffer:** A buffer of fixed size $k$ stored per pixel. It holds $k$ fragments, each containing a scalar value $v$ and a distance $d$ (fragment to eye) [S2].
*   **Pre-integrated Table:** A 3D texture (typically $128^3$ 8-bit RGBA) mapping front scalar, back scalar, and distance to color and opacity [S3, unofficial][S8].
*   **Framebuffer:** A floating-point 2D texture storing accumulated color and opacity [S3, unofficial][S8].
*   **Vertex Data:** Vertices are marked as internal or boundary to handle mesh boundaries correctly [S3, S4, unofficial].

### Pseudocode

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def cpu_sort_faces(mesh_faces, camera_position):
    """Sorts mesh faces by centroid distance to camera using radix/counting sort."""
    def float_to_sortable_int(f):
        # Converts float to sortable unsigned int preserving order
        return f * (-(f >> 31) | 0x80000000)
    
    for face in mesh_faces:
        face.centroid_int = float_to_sortable_int(face.centroid.z)
        
    # Standard LSD radix sort or counting sort on 32-bit integers
    sorted_faces = sorted(mesh_faces, key=lambda f: f.centroid_int)
    return sorted_faces

def gpu_fragment_shader(fragment, k_buffer_entries, pre_integrated_table, framebuffer):
    """
    Fragment program logic for HAVS.
    fragment: (scalar, distance)
    k_buffer_entries: list of k (scalar, distance) tuples
    """
    # Combine current fragment with k-buffer entries
    candidates = [fragment] + k_buffer_entries
    
    # Find nearest (min distance) and second nearest fragment
    # Implemented via sorting network logic equivalent to selection
    sorted_candidates = sorted(candidates, key=lambda c: c.distance)
    nearest = sorted_candidates[0]
    second_nearest = sorted_candidates[1]
    
    # 3D texture lookup for pre-integrated transfer function
    # Inputs: front_scalar, back_scalar, distance_segment
    front_scalar = nearest.scalar
    back_scalar = second_nearest.scalar
    distance_segment = second_nearest.distance - nearest.distance
    
    color, opacity = pre_integrated_table.lookup(front_scalar, back_scalar, distance_segment)
    
    # Nullify if scalar value is invalid (< 0)
    if front_scalar < 0:
        color, opacity = 0.0, 0.0
        
    # Read framebuffer color/opacity
    fb_color, fb_opacity = framebuffer.read()
    
    # Early termination check
    if fb_opacity >= 0.99:
        return # Early termination
        
    # Front-to-back compositing
    new_color = color * (1.0 - fb_opacity) + fb_color
    new_opacity = opacity * (1.0 - fb_opacity) + fb_opacity
    framebuffer.write(new_color, new_opacity)
    
    # Update k-buffer: remove nearest, keep remaining k
    remaining = [c for i, c in enumerate(candidates) if c is not nearest]
    k_buffer_entries = remaining[:len(k_buffer_entries)]
```

### Parameters and Formulas
*   **k-buffer size ($k$):** The number of fragments stored per pixel. Small $k$ (e.g., 2 or 6) is used for efficiency [S3, unofficial][S8].
*   **Integer Conversion for Sorting:** To sort floating-point coordinates using integer-based radix/counting sorts, the formula is:
    $$f_{int} = f \times (-(f \gg 31) \mid 0x80000000)$$
    where $f$ is the floating-point coordinate, $\gg$ is bitwise right shift, and $\mid$ is bitwise OR [S2].
*   **Comparison Cost (Unsorted k-buffer):** For a screen of $m \times n$ pixels, with $s_j$ fragments at pixel $j$, and buffer size $k$, the total comparisons $N_1$ are:
    $$N_1 = \sum_{j=1}^{m \times n} \left( \sum_{l=2}^k (2l-3) + (k + k - 1) \times (s_j - k) \right)$$
    The average comparisons per pixel $e_1$ is:
    $$e_1 = \frac{N_1}{m \times n}$$
    [S2]

### Edge Cases and Pitfalls
*   **Sorting Errors:** If fragments are highly out of order, the k-buffer may miss the true nearest fragments, leading to incorrect compositing and visible artifacts [S1, S2].
*   **Hardware Support:** Requires advanced read/write framebuffer access (e.g., ATI/NVIDIA extensions), which was historically considered unstable but functional [S4, S5, unofficial].
*   **Perspective Interpolation:** Texture coordinates for framebuffer lookup must be computed from window position in the fragment shader to avoid perspective interpolation errors [S3, unofficial].
*   **Invalid Scalars:** Fragments with scalar values $< 0$ should nullify the transfer function lookup result [S3, unofficial].

## Subtopics

### [HAVS with Dynamic LOD](4110-havs-with-dynamic-lod.md)
An algorithm for dynamic Level of Detail (LOD) that builds on [Hardware Assisted Visibility Sorting (HAVS)](2918-hardware-assisted-visibility-sorting-havs.md), separating boundary and internal faces, sorting internal faces by importance, and dynamically adjusting the number of faces drawn based on frame rate. It solves the problem of maintaining interactive frame rates on large tetrahedral datasets by progressively streaming approximations that increase in quality during idle frames. Unlike [Optimized HAVS (Sorted k-buffer)](4735-optimized-havs-sorted-k-buffer.md), which focuses on sorting optimization, this method focuses on adaptive primitive reduction to balance interactivity and visual fidelity.

### [Optimized HAVS (Sorted k-buffer)](4735-optimized-havs-sorted-k-buffer.md)
Optimized HAVS (Sorted k-buffer) is an optimization of the Hardware Assisted Visibility Sorting (HAVS) algorithm that maintains the k-buffer in a sorted state (ascending order of fragment distance) to reduce the computational cost of fragment-level visibility sorting. While standard HAVS requires $2k-1$ comparisons to identify the two nearest fragments for compositing, this method reduces that requirement to $k$ comparisons per fragment after the buffer is full. It is primarily used for simulating accurate radiographs involving attenuation and attenuation-emission cases, leveraging modern graphics hardware capabilities.

### [Radiography of Emissive Objects via HAVS](10257-radiography-of-emissive-objects-via-havs.md)
This method extends Hardware Assisted Visibility Sorting (HAVS) to perform radiography of emissive objects by solving the full radiative transport equation for multiple energy groups. It differs from standard HAVS by computing the transport equation directly in the fragment shader rather than using a pre-computed lookup table, which avoids errors caused by the large dynamic range of extinction coefficients and source terms. It builds on [Hardware Assisted Visibility Sorting (HAVS)](2918-hardware-assisted-visibility-sorting-havs.md) by adding per-energy-group opacity and source term handling.

## Operating characteristics
*   **Time Complexity:** Object-space sorting is $O(n+k)$ using radix/counting sort, where $n$ is the number of primitives and $k$ is the buffer size [S2]. Image-space sorting is $O(1)$ per fragment relative to $k$ (fixed depth sorting network), but the constant factor depends on $k$ [S3, unofficial].
*   **Memory Complexity:** Low memory overhead as it requires no topological information [S3, unofficial][S8]. Memory usage is dominated by the k-buffer (per-pixel) and the pre-integrated table ($128^3$) [S3, unofficial][S8].
*   **Performance:** On an ATI Radeon 9800 with a $512^2$ viewport and $k=2$, HAVS achieved up to 9.71 FPS (2331K tets/sec) for the F117 dataset (240,122 cells) [S3, unofficial]. With $k=6$, performance dropped to 4.42 FPS (1062K tets/sec) for the same dataset [S3, unofficial]. Total time (CPU+GPU) for F117 was 148ms [S3, unofficial].
*   **Strengths:** Handles arbitrary non-convex meshes with low memory overhead; simpler and more robust than PT/VO or ray casting; no preprocessing required; supports dynamic LOD and transfer function changes [S3, S4, unofficial][S8].
*   **Limitations:** Requires hardware support for read/write framebuffers; sorting errors can cause artifacts if fragments are not nearly sorted; limited by k-buffer size for highly complex scenes [S1, S2][S4, unofficial].

## Background
Developed by Callahan et al. [S3, S4, unofficial], HAVS was introduced to address the limitations of previous unstructured volume rendering techniques, which either required complex preprocessing (like PT/VO) or suffered from poor performance on GPUs [S4, S5, unofficial]. HAVS leverages the parallelism of modern GPUs by shifting the heavy lifting of fragment sorting to the GPU while using the CPU for a quick, approximate object-space sort [S3, unofficial][S8]. It is considered one of the most important unstructured grid volume rendering algorithm frameworks based on GPU [S2].

## Sources
[S1] Sorting-free pre-integrated projected tetrahedra, https://doi.org/10.1145/1838544.1838546, fetched 2026-11-01
[S2] Optimization and Implementation of Unstructured Grid Volume Rendering Algorithm, https://doi.org/10.2991/iccasm.2012.373, fetched 2026-10-26
[S3] THE K-BUFFER AND ITS APPLICATIONS TO VOLUME RENDERING, https://ctsilva.github.io/thesis/steve-callahan-ms-thesis.pdf, fetched 2026-10-27 (unofficial)
[S4] PDF A Survey of GPU-Based Volume Rendering of Unstructured Grids, https://www.sci.utah.edu/~stevec/papers/gputet_survey.pdf, fetched 2026-10-27 (unofficial)
[S5] A Survey of GPU-Based Volume Rendering of Unstructured Grids., https://www.researchgate.net/publication/220162223_A_Survey_of GPU-Based Volume Rendering of Unstructured Grids, fetched 2026-10-28 (unofficial)
[S6] PDF Parallel Unstructured Volume Rendering in ParaView, https://www.kennethmoreland.com/documents/ParallelVolRen.pdf, fetched 2026-10-27
[S7] Volume rendering for curvilinear and unstructured grids, https://www.researchgate.net/publication/4023713_Volume_rendering_for_curvilinear_and_unstructured_grids, fetched 2026-10-30 (unofficial)
[S8] PDF Hardware-Assisted Visibility Sorting for Unstructured Volume Rendering, https://www.sci.utah.edu/~stevec/papers/havs.pdf, fetched 2026-10-29
