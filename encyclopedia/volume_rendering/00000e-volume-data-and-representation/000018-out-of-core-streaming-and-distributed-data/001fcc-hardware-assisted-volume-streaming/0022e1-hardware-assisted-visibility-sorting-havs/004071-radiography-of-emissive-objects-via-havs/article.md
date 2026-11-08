# Radiography of Emissive Objects via HAVS

Also called: HAVS extension, K-buffer radiography, Transport equation radiography

## Overview
This method extends Hardware-Assisted Visibility Sorting (HAVS) to perform radiography of emissive objects by solving the full radiative transport equation for multiple energy groups [S1, unofficial]. It differs from standard HAVS by computing the transport equation directly in the fragment shader rather than using a pre-computed lookup table, which avoids errors caused by the large dynamic range of extinction coefficients and source terms [S1, unofficial]. It builds on [Hardware Assisted Visibility Sorting (HAVS)](2918-hardware-assisted-visibility-sorting-havs.md) by adding per-energy-group opacity and source term handling.

## How it works
**Inputs:**
- A tetrahedral mesh where each cell has piecewise constant extinction coefficients ($K_\nu$) and source terms ($S_\nu$) for $N$ energy groups.
- View direction.
- Number of energy groups to visualize ($N \le 4$).

**Outputs:**
- A radiograph image with $N$ channels (e.g., R, G, B, A), where each channel represents the integrated radiance for one energy group.

**Data Structures:**
1.  **Extended Face List:** A list of unique triangles from the tetrahedral mesh. Vertices are duplicated so that each face has its own set of vertex attributes, ensuring constant source and opacity values across the face [S1, unofficial].
2.  **Face Attributes:** Each face stores:
    -   Centroid position (for object-space depth sorting).
    -   Normal vector.
    -   For each energy group $g \in \{1, \dots, N\}$:
        -   Opacity term ($o_g$).
        -   Source term ($s_g$).
    -   Since HAVS operates on unique faces without connectivity, each face stores terms for the cell it belongs to and potentially a neighboring cell if it is not on the boundary [S1, unofficial].
3.  **K-Buffer:** A fragment stream sorting network implemented on GPU hardware. Each entry in the k-buffer stores:
    -   Distance from viewpoint ($d$).
    -   Opacity ($o_g$) and Source ($s_g$) for each energy group $g$.
    -   The size $k$ is determined by the number of available Multiple Render Targets (MRTs). With 3 MRTs plus framebuffer, up to 6 entries are possible, but this method uses $k=4$ to balance quality and performance [S1, unofficial].

**Steps:**

1.  **Preprocessing (Object-Space):**
    -   Duplicate vertices shared by multiple faces to ensure constant attributes per face [S1, unofficial].
    -   For each face, determine which energy group terms to use based on the view direction. Compute the dot product of the face normal $\mathbf{n}$ and the view direction $\mathbf{v}$. If the dot product indicates the ray is entering the cell, use the terms for the current cell; otherwise, use the terms for the neighboring cell (if applicable) [S1, unofficial].
    -   Sort the unique faces by their centroid depth in object space.

2.  **Rasterization and K-Buffer Management:**
    -   Pass the sorted faces to the GPU.
    -   Rasterize each face. For each fragment generated:
        -   Compute the distance $d$ from the viewpoint.
        -   Retrieve $o_g$ and $s_g$ for each energy group $g$ from the face attributes.
        -   Insert the fragment into the k-buffer for its pixel. The k-buffer maintains the $k$ fragments farthest from the viewpoint (back-to-front order) [S1, unofficial].

3.  **Fragment Program (Transport Equation Solution):**
    -   When the k-buffer is updated or a new fragment is inserted, the two entries farthest from the viewpoint are processed for back-to-front compositing.
    -   For each energy group $g$, the transport equation is solved using a first-order recurrence over the piecewise constant approximation. The radiance $L_g$ is updated as:
        $$ L_g^{new} = L_g^{old} e^{-\tau_g} + S_g (1 - e^{-\tau_g}) $$
        where $\tau_g$ is the optical depth for group $g$, derived from the extinction coefficient and distance. The source term $S_g$ and opacity $o_g$ are used directly in the fragment program [S1, unofficial].
    -   The resulting color and opacity for each energy group are composited into the framebuffer using Multiple Render Targets (MRT) [S1, unofficial].

4.  **Finalization:**
    -   After all faces are rasterized, the framebuffer contains the final radiograph with one channel per energy group.

**Parameters:**
-   $k$: K-buffer size. Set to 4 for this method to allow 4 energy groups with 3 values per fragment (distance, opacity, source) [S1, unofficial].
-   Number of energy groups: Limited to 4 (one per image channel R, G, B, A) [S1, unofficial].

**Edge Cases and Pitfalls:**
-   **Lookup Table Error:** Using a pre-computed lookup table for emissive objects introduces considerable error due to the large dynamic range of extinction coefficients and source terms. The transport equation must be computed in the fragment program [S1, unofficial].
-   **Vertex Consistency:** If vertices are not duplicated, shared vertices may have inconsistent energy group terms across faces, leading to visual artifacts. Preprocessing to duplicate vertices is required [S1, unofficial].
-   **Memory Limits:** The k-buffer size is limited by hardware MRT availability. Increasing $k$ improves image quality but reduces performance [S1, unofficial].

## Subtopics

### [Hardware-Accelerated Volume Rendering (HAVS)](10261-hardware-accelerated-volume-rendering-havs.md)
A GPU-based algorithm that computes the transport equation directly in the fragment program using piecewise constant extinction coefficients and source terms for multiple energy groups, rather than relying on a pre-computed lookup table. It allows for competitive performance with software raytracers for large numbers of energy groups and high-resolution images, scaling nearly independently of image size until card memory limits are reached.

## Operating characteristics
-   **Time Complexity:** Dominated by rasterization and k-buffer sorting. The object-space sort is $O(F \log F)$ where $F$ is the number of faces. The GPU processing is roughly linear in the number of fragments, with k-buffer operations being constant time per fragment for fixed $k$.
-   **Memory Complexity:** Proportional to $k \times W \times H \times C$, where $W, H$ are image dimensions, $C$ is the number of channels (energy groups), and $k$ is the k-buffer size. With $k=4$ and 4 energy groups, this is significant but manageable on modern GPUs [S1, unofficial].
-   **Performance:** Scales nearly independently of image size until card memory limits are reached. Competitive with software raytracers for large numbers of energy groups and high-resolution images [S1, unofficial].
-   **Strengths:** Reduces the number of primitives rasterized compared to Projected Tetrahedra by operating on unique faces without connectivity information [S1, unofficial]. Handles emissive objects accurately by solving the transport equation in the fragment program.
-   **Limitations:** Limited to 4 energy groups due to hardware MRT constraints. Requires preprocessing to duplicate vertices. The k-buffer size limits the accuracy of depth sorting for highly complex scenes.

## Background
This method was developed for simulating radiographs of subjects with emissive materials for laser experiments, such as those at the National Ignition Facility (NIF) [S1, unofficial]. It extends the HAVS algorithm proposed by Callahan et al. [2] to handle the full transport equation for emissive objects. The approach is particularly relevant for inertial confinement fusion (ICF) diagnostics, where accurate simulation of radiographs is crucial for designing experiments [S1, unofficial].

## Sources
[S1] Hardware-Accelerated Simulated Radiography, https://digital.library.unt.edu/ark:/67531/metadc876978/?q=front-to-back compositing, fetched 2026-10-29 (unofficial)
