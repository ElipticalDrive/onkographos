# Radiography of Emissive Objects via HAVS

Also called: HAVS extension, K-buffer radiography, Transport equation radiography

## Overview
This method extends Hardware-Assisted Visibility Sorting (HAVS) to perform radiography of emissive objects by solving the full radiative transport equation for multiple energy groups [S1, unofficial]. It differs from standard HAVS by computing the transport equation directly in the fragment shader rather than using a pre-computed lookup table, which avoids errors caused by the large dynamic range of extinction coefficients and source terms [S1, unofficial]. It builds on [Hardware Assisted Visibility Sorting (HAVS)](2918-hardware-assisted-visibility-sorting-havs.md) by adding per-energy-group opacity and source term handling [S1, unofficial].

## How it works
**Inputs:**
- A tetrahedral mesh where each cell has piecewise constant extinction coefficients ($K_\nu$) and source terms ($S_\nu$) for $N$ energy groups [S1, unofficial].
- View direction [S1, unofficial].
- Number of energy groups to visualize ($N \le 4$) [S1, unofficial].

**Outputs:**
- A radiograph image with $N$ channels (e.g., R, G, B, A), where each channel represents the integrated radiance for one energy group [S1, unofficial].

**Data Structures:**
1.  **Extended Face List:** A list of unique triangles from the tetrahedral mesh. Vertices are duplicated so that each face has its own set of vertex attributes, ensuring constant source and opacity values across the face [S1, unofficial].
2.  **Face Attributes:** Each face stores centroid position (for object-space depth sorting), normal vector, and for each energy group $g \in \{1, \dots, N\}$: opacity term ($o_g$) and source term ($s_g$) [S1, unofficial]. Since HAVS operates on unique faces without connectivity, each face stores terms for the cell it belongs to and potentially a neighboring cell if it is not on the boundary [S1, unofficial].
3.  **K-Buffer:** A fragment stream sorting network implemented on GPU hardware. Each entry stores distance from viewpoint ($d$), opacity ($o_g$) and source ($s_g$) for each energy group $g$ [S1, unofficial]. The size $k$ is determined by the number of available Multiple Render Targets (MRTs). With 3 MRTs plus framebuffer, up to 6 entries are possible, but this method uses $k=4$ to balance quality and performance [S1, unofficial].

**Steps:**

1.  **Preprocessing (Object-Space):**
    - Duplicate vertices shared by multiple faces to ensure constant attributes per face [S1, unofficial].
    - For each face, determine which energy group terms to use based on the view direction. Compute the dot product of the face normal $\mathbf{n}$ and the view direction $\mathbf{v}$ [S1, unofficial]. If the dot product indicates the ray is entering the cell, use the terms for the current cell; otherwise, use the terms for the neighboring cell (if applicable) [S1, unofficial].
    - Sort the unique faces by their centroid depth in object space [S1, unofficial].

2.  **Rasterization and K-Buffer Management:**
    - Pass the sorted faces to the GPU [S1, unofficial].
    - Rasterize each face. For each fragment generated:
        - Compute the distance $d$ from the viewpoint [S1, unofficial].
        - Retrieve $o_g$ and $s_g$ for each energy group $g$ from the face attributes [S1, unofficial].
        - Insert the fragment into the k-buffer for its pixel. The k-buffer maintains the $k$ fragments farthest from the viewpoint in back-to-front order [S1, unofficial].

3.  **Fragment Program (Transport Equation Solution):**
    - When the k-buffer is updated or a new fragment is inserted, the two entries farthest from the viewpoint are processed for back-to-front compositing [S1, unofficial].
    - For each energy group $g$, the transport equation is solved using a first-order recurrence over the piecewise constant approximation [S1, unofficial].
    - MRTs are used to keep k-buffer entries during rasterization, not for final compositing [S1, unofficial].

4.  **Finalization:**
    - After all faces are rasterized, the framebuffer contains the final radiograph with one channel per energy group [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def radiography_via_havs(mesh, view_dir, num_groups=4):
    # 1. Preprocessing
    faces = duplicate_shared_vertices(mesh.unique_faces)
    for face in faces:
        dot_val = dot(face.normal, view_dir)
        if dot_val > 0:  # Ray entering cell
            face.o = face.cell_opacity
            face.s = face.cell_source
        else:            # Ray exiting or neighbor cell
            face.o = face.neighbor_opacity
            face.s = face.neighbor_source
    faces.sort_by_centroid_depth()

    # 2. Rasterization & K-Buffer
    k_buf = KBuffer(capacity=4, channels=num_groups)
    for face in faces:
        for frag in rasterize(face):
            frag.d = distance(frag.position, viewpoint)
            frag.o = face.o
            frag.s = face.s
            k_buf.insert(frag)

    # 3. Fragment Program
    for entry in k_buf.entries:
        for g in range(num_groups):
            tau = entry.o[g] * entry.d
            L_new = L_old * exp(-tau) + entry.s[g] * (1 - exp(-tau))
            L_old = L_new
        k_buf.update_remaining(entry)

    return framebuffer
```

**Parameters:**
- $k$: K-buffer size. Set to 4 for this method to allow 4 energy groups with 3 values per fragment (distance, opacity, source) [S1, unofficial].
- Number of energy groups: Limited to 4 (one per image channel R, G, B, A) [S1, unofficial].

**Edge Cases and Pitfalls:**
- **Lookup Table Error:** Using a pre-computed lookup table for emissive objects introduces considerable error due to the large dynamic range of extinction coefficients and source terms. The transport equation must be computed in the fragment program [S1, unofficial].
- **Vertex Consistency:** If vertices are not duplicated, shared vertices may have inconsistent energy group terms across faces, leading to visual artifacts. Preprocessing to duplicate vertices is required [S1, unofficial].
- **Memory Limits:** The k-buffer size is limited by hardware MRT availability. Increasing $k$ improves image quality but reduces performance [S1, unofficial].

## Subtopics

### [Hardware-Accelerated Volume Rendering (HAVS)](10261-hardware-accelerated-volume-rendering-havs.md)
Hardware-Accelerated Volume Rendering (HAVS) is a GPU-based algorithm for simulating radiographs that accounts for both attenuation and emission, solving the full transport equation for emissive objects. It extends the parent method by computing the transport equation directly in the fragment program using piecewise constant extinction coefficients and source terms for multiple energy groups, rather than relying on a pre-computed lookup table. This approach scales nearly independently of image size until card memory limits are reached.

## Operating characteristics
-   **Time Complexity:** Dominated by rasterization and k-buffer sorting. The object-space sort is $O(F \log F)$ where $F$ is the number of faces. The GPU processing is roughly linear in the number of fragments, with k-buffer operations being constant time per fragment for fixed $k$ [S1, unofficial].
-   **Memory Complexity:** Proportional to $k \times W \times H \times C$, where $W, H$ are image dimensions, $C$ is the number of channels (energy groups), and $k$ is the k-buffer size. The method uses $k=4$ to balance quality and performance, fitting within the three MRTs plus framebuffer limit [S1, unofficial].
-   **Performance:** Scales nearly independently of image size until card memory limits are reached [S1, unofficial].
-   **Strengths:** Reduces the number of primitives rasterized compared to Projected Tetrahedra by operating on unique faces without connectivity information [S1, unofficial]. Handles emissive objects accurately by solving the transport equation in the fragment program [S1, unofficial].
-   **Limitations:** Limited to 4 energy groups due to hardware MRT constraints. Requires preprocessing to duplicate vertices. The k-buffer size limits the accuracy of depth sorting for highly complex scenes [S1, unofficial].

## Background
This method was developed for simulating radiographs of subjects with emissive materials for laser experiments, such as those at the National Ignition Facility (NIF) [S1, unofficial]. It extends the HAVS algorithm proposed by Callahan et al. to handle the full transport equation for emissive objects [S1, unofficial]. The approach is particularly relevant for inertial confinement fusion (ICF) diagnostics, where accurate simulation of radiographs is crucial for designing experiments [S1, unofficial].

## Sources
[S1] Hardware-Accelerated Simulated Radiography, https://digital.library.unt.edu/ark:/67531/metadc876978/?q=front-to-back compositing, fetched 2026-10-29 (unofficial)
