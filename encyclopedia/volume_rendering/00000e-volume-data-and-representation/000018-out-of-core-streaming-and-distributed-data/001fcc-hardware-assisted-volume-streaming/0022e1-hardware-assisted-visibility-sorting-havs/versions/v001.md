# Hardware Assisted Visibility Sorting (HAVS)


## Overview
Hardware Assisted Visibility Sorting (HAVS) is a volume rendering algorithm for unstructured grids that combines object-space and image-space processing to handle arbitrary nonconvex meshes with low memory overhead [S2, unofficial]. It solves the visibility sorting problem for unstructured data by using the CPU to perform a partial sort of primitives and the GPU to complete the sorting and compositing via a fixed-depth k-buffer [S2, unofficial]. Unlike HAPT, which relies on centroid sorting, HAVS provides more accurate visibility sorting by incrementally sorting the fragment stream on the GPU [Parent]. It is distinct from pure object-space methods like ZSWEEP, which suffer from lower sorting efficiency and higher GPU burden [S1].

## How it works
HAVS operates in two phases: object-space partial sorting on the CPU and image-space complete sorting/compositing on the GPU [S2, unofficial]. The method relies exclusively on face information rather than explicit cell topology, requiring no topological information and resulting in very low memory overhead [S2, S3, unofficial].

### Inputs and Outputs
*   **Input:** Unstructured mesh (e.g., tetrahedral cells) with scalar values at vertices; viewing parameters (MVP matrix, ModelView matrix); transfer function (3D pre-integrated table).
*   **Output:** Composited color and opacity image in the framebuffer.

### Data Structures
*   **k-buffer:** A buffer of length $k$ associated with each pixel. It stores $k$ fragments. Each fragment contains a scalar value $v$ and the distance $d$ from the eye (viewport) [S1]. In the reference implementation, $k=6$, storing 6 fragments plus the current incoming fragment for comparison [S2, unofficial].
*   **Textures:**
    *   Tex 0: Framebuffer (2D RGBA, 16/32-bit float) [S2, unofficial].
    *   Tex 1, 2, 3: k-buffer entries. Each texture stores 2 entries (scalar and distance) [S2, unofficial].
    *   Tex 4: 3D pre-integrated transfer function (RGBA, 8/16-bit int) [S2, unofficial].
*   **Parameters:**
    *   `mvp[4]`: Model-View-Projection matrix (4x4 floats) [S2, unofficial].
    *   `mv[4]`: ModelView matrix (4x4 floats) [S2, unofficial].
    *   `sz`: Scale and bias parameters $\{1/p_w, 1/p_h, (1-1/z_{size})/max\_len, 1/(2 \cdot z_{size})\}$ for texture coordinate normalization [S2, unofficial].
    *   `e`: Eye position vector [S2, unofficial].

### Algorithm Steps

#### 1. Object-Space Sorting (CPU)
1.  Mark vertices as internal or boundary to handle mesh boundaries correctly [S2, unofficial].
2.  Compute the centroid for each face of the unstructured mesh (e.g., tetrahedra) [S2, unofficial].
3.  Sort the faces based on their centroids using a floating-point radix sort (LSD radix sort combined with counting sort) [S1][S2, unofficial].
    *   To sort floating-point coordinates, convert them to 32-bit unsigned integers using the bitwise formula:
        $$f = f \times (-(f>>31) | 0x80000000)$$
        where $f$ is the float, `>>` is bitwise right shift, and `|` is bitwise OR [S1].
    *   Divide the resulting integers into four 8-bit blocks and use counting sort for each block [S1].
4.  Send the sorted faces to the GPU for rasterization [S2, unofficial].

#### 2. Image-Space Processing (GPU)
The GPU rasterizes the sorted faces. For each fragment generated, the following fragment program steps are executed:

1.  **Transformations:**
    *   Transform vertex to clip coordinates: $oPos = MVP \times iPos$ [S2, unofficial].
    *   Transform vertex to eye coordinates: $oTex1 = MV \times iPos$ [S2, unofficial].
    *   Pass scalar data value: $oTex0 = iTex0$ [S2, unofficial].

2.  **Framebuffer Lookup:**
    *   Compute texture coordinates from window position to avoid perspective interpolation errors: $t = p \times sz$ [S2, unofficial].
    *   Lookup current framebuffer color/opacity: $c_0 = \text{Tex2D}(\text{Tex0}, t)$ [S2, unofficial].

3.  **Early Termination:**
    *   If framebuffer opacity $c_0.w \ge 0.99$, kill the fragment [S2, unofficial].

4.  **Load k-buffer:**
    *   Load existing k-buffer entries from textures into temporaries $a_1$ through $a_6$ [S2, unofficial].
    *   Compute the current fragment's distance to the eye: $a_0.y = \text{distance}(eye, \text{fragment})$ and scalar $a_0.x = v.x$ [S2, unofficial].

5.  **Find Nearest Fragment ($r_0$):**
    *   Use a sorting network to find the fragment with the minimum distance among the current fragment ($a_0$) and the 6 k-buffer entries ($a_1 \dots a_6$).
    *   Example logic for comparing two entries $x$ and $y$ with distances $x.y, y.y$:
        ```python
        diff = x.y - y.y
        min_frag = (diff < 0) ? x : y
        max_frag = (diff < 0) ? y : x
        ```
    *   Iteratively compare $r_0$ with $a_1 \dots a_6$ to find the global minimum $r_0$ and store the rest in $r_1 \dots r_6$ [S2, unofficial].

6.  **Find Second Nearest Fragment ($r_7$):**
    *   Find the minimum distance among the remaining fragments $r_1 \dots r_6$ using the same sorting network logic. The result is $r_7$ [S2, unofficial].

7.  **Transfer Function Lookup:**
    *   Set up coordinates for the 3D pre-integrated table:
        *   Front scalar: $t.x = r_0.x$
        *   Back scalar: $t.y = r_7.x$
        *   Distance: $t.z = (r_7.y - r_0.y) \times sz.z + sz.w$ [S2, unofficial].
    *   Lookup color/opacity: $c = \text{Tex3D}(\text{Tex4}, t)$ [S2, unofficial].

8.  **Nullify Invalid Entry:**
    *   If the winning fragment's scalar value $r_0.x < 0$, nullify the color/opacity $c$ [S2, unofficial].

9.  **Composite:**
    *   Perform front-to-back blending with the framebuffer:
        $$result.color[0] = c \times (1 - c_0.w) + c_0$$ [S2, unofficial].

10. **Write Back k-buffer:**
    *   Write the remaining $k$ fragments (excluding the two used for compositing) back to the output buffers. In the reference implementation, entries $r_1, r_3, r_5$ are written to `result.color[1..3]` after reordering [S2, unofficial].

### Edge Cases and Pitfalls
*   **Out-of-Order Fragments:** If fragments are highly out of order, the k-buffer (which only considers $k$ fragments) may produce incorrect rendering results because it cannot resolve visibility beyond its depth [S1].
*   **Perspective Interpolation:** Texture coordinates for framebuffer lookup must be computed from the window position in the fragment shader, not from interpolated values, to avoid errors [S2, unofficial].
*   **Sorting Depth Trade-off:** The depth of the fragment sorting network can be varied. Increasing the depth allows the use of a less accurate object-space sorting algorithm, shifting work from CPU to GPU [S2, unofficial].

## Subtopics

### [HAVS with Dynamic LOD](4110-havs-with-dynamic-lod.md)
An algorithm for dynamic LOD that builds on HAVS, separating boundary and internal faces, sorting internal faces by importance, and dynamically adjusting the number of faces drawn based on frame rate.

### [Optimized HAVS (Sorted k-buffer)](4735-optimized-havs-sorted-k-buffer.md)
An optimization of HAVS that keeps the k-buffer sorted in ascending order of fragment distance to decrease comparison time in the fragment-level sorting process.

## Operating characteristics
*   **Complexity:** Object-space sorting complexity is $O(n+k)$, where $n$ is the number of primitives and $k$ is the k-buffer size [S1]. The image-space comparison cost per pixel $j$ with $s_j$ fragments is $N_1 = \sum_{j=1}^{m \times n} (\sum_{l=2}^k (2l-3) + (2k-1)(s_j - k))$ [S1].
*   **Performance:** Tested on an ATI Radeon 9800 with a 512x512 viewport and 128^3 transfer function [S2, unofficial].
    *   With $k=2$, the GPU portion achieves 9.71 fps (F117 dataset) to 2.07 fps (Spx2 dataset) [S2, unofficial].
    *   With $k=6$, performance drops to 4.42 fps (F117) to 1.70 fps (Spx2) due to increased sorting comparisons [S2, unofficial].
    *   Total time (CPU+GPU, $k=2$) ranges from 148ms (F117) to 773ms (Fighter) [S2, unofficial].
    *   With early ray termination and high opacity, over 6 million cells/sec were achieved on the Fighter dataset [S2, unofficial].
*   **Strengths:** Handles arbitrary nonconvex meshes; low memory overhead (no topological info); maximum data size bounded by main memory, not texture memory; simple implementation (<200 lines of rendering code) [S2, unofficial].
*   **Limitations:** Rendering accuracy depends on $k$; small $k$ reduces precision for highly out-of-order fragments; performance decreases with larger $k$ due to sorting network depth [S1][S2, unofficial].
*   **When to use:** Interactive visualization of large unstructured volumetric datasets where dynamic LOD or transfer function changes are needed [S2, unofficial].

## Background
HAVS was introduced by Steven Callahan, Milan Ikits, Joao Comba, and Claudio Silva in their 2005 thesis "The K-Buffer and Its Applications to Volume Rendering" [S2, unofficial]. It was developed to address the limitations of previous unstructured grid rendering methods, offering a hybrid CPU/GPU approach that is simpler and more efficient than pure object-space methods like ZSWEEP [S1]. The method is particularly noted for its simplicity and adaptability to dynamic LOD and isosurface extraction [S2, S3, unofficial].

## Sources
[S1] Optimization and Implementation of Unstructured Grid Volume Rendering Algorithm, https://doi.org/10.2991/iccasm.2012.373, fetched 2026-10-26
[S2] THE K-BUFFER AND ITS APPLICATIONS TO VOLUME RENDERING, https://ctsilva.github.io/thesis/steve-callahan-ms-thesis.pdf, fetched 2026-10-27 (unofficial)
[S3] Volume rendering for curvilinear and unstructured grids, https://www.researchgate.net/publication/4023713_Volume_rendering_for_curvilinear_and_unstructured_grids, fetched 2026-10-30 (unofficial)
