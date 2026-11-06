# Shear-Warp Algorithm

Also called: Lacroute-Levoy algorithm, Lacroute and Levoy algorithm, Shear-Warp Factorization

## Overview
The Shear-Warp Algorithm is a fast, software-based volume rendering technique that reduces the number of resampling passes from three to two by factorizing the viewing transformation matrix into a 3D shear and a 2D warp [S1, S2, S12, unofficial]. It solves the problem of efficient rendering from arbitrary viewing angles by transforming the volume into a sheared object space where viewing rays are perpendicular to the data slices, allowing for fixed-weight bilinear interpolation and efficient run-length encoding (RLE) optimizations [S3, S9, unofficial]. Unlike pure ray-casting which performs trilinear interpolation per pixel, or splatting which distributes kernels, Shear-Warp exploits spatial coherence by processing slices in object order and warping the result in image order, making it historically the fastest purely software-based volume rendering algorithm for moderate datasets [S6, S12, unofficial].

## How it works
The algorithm factorizes the general viewing transformation matrix $M_{view}$ into three components: a permutation matrix $P$, a shear matrix $S$, and a 2D warp matrix $M_{warp2D}$ [S2, S12, unofficial]:
$$ M_{view} = M_{warp2D} \cdot S \cdot P $$
The process consists of three phases: Permutation/Shear, Composite, and Warp [S2, unofficial][S5, unofficial].

**Inputs and Outputs:**
*   **Input:** 3D volume data $V(x,y,z)$, viewing transformation matrix $M_{view}$, transfer function (mapping density to color/opacity), and image dimensions $(W, H)$.
*   **Output:** 2D image buffer $I(u,v)$ with color and opacity values.

**Data Structures:**
*   **Volume Data:** Stored as a 3D array or 3D texture. For optimization, three Run-Length Encoded (RLE) structures are pre-computed, one for each principal axis (X, Y, Z) [S3, S12, unofficial]. Each RLE entry contains a start index, length, and pre-classified opacity/density [S17, unofficial].
*   **Intermediate Image:** A 2D buffer of size $(W, H)$ storing accumulated color and opacity. To support early ray termination, opaque pixels store an offset (pointer) to the next non-opaque pixel in the scanline [S2, S11, unofficial].

**Numbered Steps:**

1.  **Permutation (Axis Selection):** Determine the principal viewing axis by finding the coordinate axis (X, Y, or Z) most aligned with the view direction vector $v_o$ [S12, unofficial]. Construct the permutation matrix $P$ to align this axis with the Z-axis of the intermediate coordinate system [S2, S12, unofficial].
    *   If $v_o$ is closest to Z, $P = I$ (identity).
    *   If $v_o$ is closest to X, swap X and Z axes.
    *   If $v_o$ is closest to Y, swap Y and Z axes.
    *   *Pitfall:* If the angle is near 45 degrees to any axis, artifacts increase; the choice is arbitrary but affects artifact direction [S9, unofficial][S15, book].

2.  **Shear and Resampling:** Transform the volume slices (perpendicular to the principal axis) into an intermediate 2D image space [S2, S12, unofficial].
    *   Calculate shear coefficients $s_x, s_y$ from the view direction $v_o = [v_{o,x}, v_{o,y}, v_{o,z}]^T$ in the permuted space:
        $$ s_x = -\frac{v_{o,x}}{v_{o,z}}, \quad s_y = -\frac{v_{o,y}}{v_{o,z}} $$
    *   For parallel projection, the shear matrix is:
        $$ S = \begin{pmatrix} 1 & 0 & s_x & 0 \\ 0 & 1 & s_y & 0 \\ 0 & 0 & 1 & 0 \\ 0 & 0 & 0 & 1 \end{pmatrix} $$
    *   For perspective projection, an additional scaling factor is applied to each slice to account for foreshortening, and the shear matrix includes a perspective component $s'_w$ [S2, unofficial][S15, book]:
        $$ S_{persp} = \begin{pmatrix} 1 & 0 & s'_x & 0 \\ 0 & 1 & s'_y & 0 \\ 0 & 0 & 1 & 0 \\ 0 & 0 & s'_w & 1 \end{pmatrix} $$
    *   Resample each volume slice using bilinear interpolation into the intermediate image. The shear ensures that sampling rays are perpendicular to the slices, allowing fixed interpolation weights computed once per frame [S3, S9, unofficial].
    *   *Optimization:* Use RLE to skip transparent voxel runs. Use the intermediate image's opaque-pixel offsets to skip occluded pixels [S2, S11, unofficial].

3.  **Composite:** Composite the sheared slices in front-to-back order into the intermediate image using the Porter-Duff "over" operator [S2, S12, unofficial].
    *   For each pixel $(u,v)$ in the intermediate image, accumulate color $C$ and opacity $\alpha$:
        $$ C_{new} = C_{slice} \cdot \alpha_{slice} + C_{accum} \cdot (1 - \alpha_{slice}) $$
        $$ \alpha_{new} = \alpha_{slice} + \alpha_{accum} \cdot (1 - \alpha_{slice}) $$
    *   *Early Ray Termination:* If $\alpha_{accum}$ exceeds a user-defined threshold (e.g., 0.95), stop processing further slices for that pixel [S2, S11, unofficial].

4.  **Warp:** Transform the intermediate image to the final image space using the 2D warp matrix $M_{warp2D}$ [S2, S12, unofficial].
    *   Compute $M_{warp2D}$ as:
        $$ M_{warp2D} = M_{view} \cdot S^{-1} \cdot P^{-1} $$
        (Note: Some sources define $M_{warp2D} = M_{view} \cdot M_{shear}^{-1}$ assuming $P$ is handled in the permutation step [S7, S21, unofficial]).
    *   Resample the intermediate image using bilinear interpolation to produce the final image [S2, S12, unofficial]. This step corrects the distortion introduced by the shear and applies the correct perspective/parallel projection [S22, unofficial].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def shear_warp_render(volume, M_view, transfer_function, width, height):
    # 1. Permutation
    # Determine principal axis (z, x, or y) based on view direction
    view_dir = get_view_direction(M_view)
    principal_axis = argmax(abs(view_dir)) # 0:x, 1:y, 2:z
    
    # Construct permutation matrix P to align principal axis with Z
    P = get_permutation_matrix(principal_axis)
    
    # 2. Shear
    # Calculate shear coefficients
    # Transform view direction to permuted space if necessary, 
    # but typically derived from M_view components directly
    # vo = [m12*m23 - m22*m13, m21*m13 - m11*m23, m11*m22 - m21*m12]
    vo = compute_view_direction_from_matrix(M_view)
    
    sx = -vo[0] / vo[2]
    sy = -vo[1] / vo[2]
    
    # For perspective, additional scaling factors are needed per slice
    # S_par for parallel projection
    S = [
        [1, 0, sx, 0],
        [0, 1, sy, 0],
        [0, 0, 1, 0],
        [0, 0, 0, 1]
    ]
    
    # Pre-compute RLE structures for the volume along the principal axis
    # This requires three copies of RLE data for X, Y, Z directions
    rle_data = build_rle(volume, transfer_function, principal_axis)
    
    # Initialize intermediate image with opaque pixel offsets
    intermediate_image = init_intermediate_image(width, height)
    
    # 3. Composite
    # Iterate through slices in front-to-back order
    for slice_idx in range(num_slices):
        # Get slice data from RLE or volume
        slice_data = get_slice_data(rle_data, slice_idx)
        
        # Resample slice into intermediate image using bilinear interpolation
        # The shear transforms the slice coordinates
        resample_slice_to_intermediate(slice_data, S, intermediate_image, slice_idx)
        
        # Early ray termination check can be integrated here
        if check_early_termination(intermediate_image):
            break
            
    # 4. Warp
    # Compute warp matrix
    S_inv = invert_matrix(S)
    P_inv = invert_matrix(P)
    M_warp2D = multiply_matrices(M_view, S_inv, P_inv)
    
    # Warp intermediate image to final screen
    final_image = warp_image(intermediate_image, M_warp2D, width, height)
    
    return final_image

def resample_slice_to_intermediate(slice_data, S, intermediate_image, slice_idx):
    # For each pixel in the intermediate image, map back to sheared volume coordinates
    # and sample using bilinear interpolation
    for y in range(height):
        for x in range(width):
            # Map pixel (x,y) to sheared volume coordinates
            # This involves inverse warping or forward shearing depending on implementation
            # Forward shearing is typically used: project slice vertices to intermediate space
            pass
```

## Subtopics

### [Parallel Perspective Shear-Warp Rendering](802-parallel-perspective-shear-warp-rendering.md)
A distributed technique that parallelizes the compositing step across multiple nodes using MPI, partitioning intermediate image scanlines rather than volume data. It is chosen when rendering high-resolution perspective volumes on clusters without dedicated graphics cards, avoiding OpenGL extension dependencies.

### [CPU-based Shear-Warp](1161-cpu-based-shear-warp.md)
Implements the algorithm on a CPU using 1D RLE compression along principal axes. It distinguishes itself from GPU variants by its sequential execution model and from the original proposal by relying on 1D rather than 3D data structures.

### [Permutation Proof for Projection and Warp](1702-permutation-proof-for-projection-and-warp.md)
A mathematical proof showing that the projection matrix $P$ and warp matrix $W$ can be permuted ($W \cdot P \cdot S$), enabling specific optimization paths in the compositing step by allowing the projection to precede the warp.

### [Spatially Partitioned Shear-Warp Method](3481-spatially-partitioned-shear-warp-method.md)
A parallel technique that partitions the volume into axis-aligned sub-volumes, renders each into intermediate baseplanes, and composites them via a binary tree. It allows view-independent partitioning and handles large datasets on distributed clusters with specific network optimizations.

### [Shear-warp Factorization for Combined Surface/Volume Rendering](7392-shear-warp-factorization-for-combined-surface-volume-renderi.md)
Combines volume rendering for complex organs with surface rendering for instruments (e.g., needles) in surgical simulators. It uses a shared intermediate sheared space and Z-buffer depth comparison to efficiently composite the two.

### [GPU-based Shear-Warp](7425-gpu-based-shear-warp.md)
Maps the slice-based compositing and warping steps onto programmable graphics hardware pipelines using vertex and fragment shaders. It leverages parallel shader execution for shading and compositing, differing from CPU-based variants.

### [Optimization Technique](7922-optimization-technique.md)
Introduces hardware-aware shortcuts like binary masks or dynamic scaling to reduce computational complexity, distinct from the base algorithm's transformation factorization.

### Texture-Based Warp
Uses 2D texturing hardware (OpenGL) to perform the final warp step by drawing the intermediate image as a textured quad. It is faster for large images but constrained by power-of-two texture size requirements [S24, unofficial].

### Software Warp
Computes the final 2D image using a pure software algorithm. It is faster for small images but lacks automatic object ordering for intersecting polygons [S24, unofficial].

### GPU-Accelerated On-the-Fly Deskewing
An open-source package for real-time transformation of Oblique Plane Microscopy (OPM) data, producing extended depth of field projections using GPU acceleration [S18, abstract only, unofficial].

### Resampling from Sheared Space
A specific optimization where the first non-transparent cell is found using a depth routine on a sheared mask, and bilinear interpolation is used for sampling, with binary volume skipping for empty regions [S8, unofficial].

## Operating characteristics
*   **Time Complexity:** $O(N \cdot W \cdot H)$ where $N$ is the number of slices, and $W, H$ are image dimensions. The RLE optimization reduces the effective number of samples by skipping transparent voxels and opaque pixels [S2, S11, unofficial].
*   **Memory Complexity:** $O(V + W \cdot H)$ where $V$ is the volume size. Requires storing the volume and three RLE structures (one per axis), leading to high memory overhead (up to 4x the volume size) [S9, S12, unofficial].
*   **Measured Performance:** Renders a $256^3$ voxel dataset in ~1 second for parallel projection and ~3 seconds for perspective projection on an SGI Indigo R4000 [S1, S24, unofficial]. On modern hardware, it achieves sub-second frame rates for moderate datasets [S6, unofficial].
*   **Strengths:** Extremely fast software implementation; exploits spatial coherence; supports early ray termination; simple to implement on CPUs [S2, S6, unofficial].
*   **Limitations:** Reduced image quality compared to ray-casting due to bilinear interpolation and fixed sampling rate; stair-casing artifacts near 45-degree viewing angles; perspective projection is slower due to diverging rays and many-to-one voxel-to-pixel mapping [S1, S4, S11, unofficial].
*   **When to Use:** Real-time or interactive software rendering on CPUs where hardware acceleration is unavailable or insufficient; moderate dataset sizes [S6, S12, unofficial].
*   **When Not to Use:** High-quality medical imaging requiring trilinear interpolation; very large datasets exceeding memory limits; scenarios requiring dynamic transfer functions without recomputing RLE structures [S9, S12, unofficial].

## Background
The Shear-Warp algorithm was proposed by Lacroute and Levoy in 1994 [S1, S3, S6, unofficial]. It was developed to address the computational cost of volume rendering on standard workstations, aiming to be the fastest purely software-based method [S6, unofficial]. The algorithm factorizes the viewing transformation to exploit memory alignment and fixed scaling factors, reducing the need for expensive trilinear interpolation [S3, S9, unofficial]. It has been used in systems like Volpack and influenced hardware implementations such as VolumePro [S6, unofficial][S15, book].

## Sources
[S1] Interactive volume ray tracing, https://doi.org/10.22028/d291-25931, fetched 2026-10-26 (unofficial)
[S2] Direct Volume Rendering Techniques, https://www.scribd.com/document/853597780/VIS-Modules-06-Direct-Volume-Rendering, fetched 2026-10-25 (unofficial)
[S3] Advanced illumination and view-selection techniques for volume rendering and its application to medical imaging, http://hdl.handle.net/10803/371154, fetched 2026-10-27 (unofficial)
[S4] NASA Technical Reports Server (NTRS) 19970021689: Direct Volume Rendering with Shading via Three-Dimensional Textures, https://archive.org/details/NASA_NTRS_Archive_19970021689, fetched 2026-10-26 (unofficial)
[S5] NASA Technical Reports Server (NTRS) 19970022121: Direct Volume Rendering with Shading via Three-Dimensional Textures, https://archive.org/details/NASA_NTRS_Archive_19970022121, fetched 2026-10-26 (unofficial)
[S6] Shear-warp factorization volume rendering visualization system, https://archive.org/details/0162018299360, fetched 2026-10-26 (unofficial)
[S7] GPU-Based Volume Rendering for Medical Imagery, https://inria.hal.science/inria-00461484v1, fetched 2026-10-26 (unofficial)
[S8] Interactive Volume-Rendering Techniques for Medical Data Visualization, https://doi.org/10.2312/diss.20018138, fetched 2026-10-26 (unofficial)
[S9] Volume rendering techniques for general purpose graphics hardware, https://opus4.kobv.de/opus4-fau/files/9/Dissertation.pdf, fetched 2026-10-26 (unofficial)
[S10] Real-Time Mono- and Multi-Volume Rendering of Large Medical Datasets on Standard PC Hardware, https://doi.org/10.2312/diss.20058170, fetched 2026-10-26 (unofficial)
[S11] [PDF] Direct Volume Rendering | CGL @ ETHZ, https://cgl.ethz.ch/teaching/former/scivis_07/Notes/stuff/StuttgartCourse/VIS-Modules-06-Direct_Volume_Rendering.pdf, fetched 2026-10-26 (unofficial)
[S12] PDF Shear-Warp Volume Rendering, https://www.cs.unh.edu/~cs880/lectureNotes/ShearWarp.pdf, fetched 2026-10-27 (unofficial)
[S13] Volume Ray Casting Techniques and Applications using ..., https://halogenica.net/bin/graphics/volume-rendering-using-gpgpu/Thesis.pdf, fetched 2026-10-27 (unofficial)
[S14] Volume Visualization: A Technical Overview with a Focus on ..., https://pmc.ncbi.nlm.nih.gov/articles/PMC3138940/, fetched 2026-10-27
[S15] PDF Direct Volume Rendering, https://www.sci.utah.edu/~stevec/classes/cs5630/dvr.pdf, fetched 2026-10-27 (book)
[S16] (PDF) EWA volume splatting - ResearchGate, https://www.researchgate.net/publication/48546966_EWA_volume_splatting, fetched 2026-10-28 (unofficial)
[S17] Medical Volume Rendering Techniques, https://arxiv.org/pdf/1802.07710v1, fetched 2026-10-27 (unofficial)
[S18] Fast volume rendering using a shear-warp factorisation ... - ResearchGate, https://www.researchgate.net/publication/243763776_Fast_volume_rendering_using_a_shear-warp_factorisation_of_the_viewing_transform, fetched 2026-10-27 (abstract only, unofficial)
[S19] Fast Volume Rendering Using a Shear-Warp Factorization of (unofficial)
        the Viewing Transform [#LaLe##1#], https://www.cg.tuwien.ac.at/studentwork/VisFoSe98/ak/FoSe_Notes/node46.html, fetched 2026-10-27 (unofficial)
[S20] Volume ray casting techniques and applications using general purpose computations on graphics processing units, https://repository.rit.edu/cgi/viewcontent.cgi?article=6466&context=theses, fetched 2026-10-28 (unofficial)
[S21] GPU-Based Volume Rendering for Medical Imagery, https://scholarly.org/pdf/display/gpu-based-volume-rendering-for-medical-imagery, fetched 2026-10-28 (unofficial)
[S22] The Shear-Warp algorithm. Rendering a sliced volume is replaced by ..., https://www.researchgate.net/figure/The-Shear-Warp-algorithm-Rendering-a-sliced-volume-is-replaced-by-shearing-the-volume_fig1_228912978, fetched 2026-10-28 (unofficial)
[S23] Shear Warp, https://www.cg.tuwien.ac.at/courses/Visualisierung/2002-2003/Beispiel1/2-BrucknerSt-SeemannR/index.htm, fetched 2026-10-28 (unofficial)
[S24] The Perspective Shear-Warp Algorithm in a Virtual Environment., https://www.researchgate.net/publication/220943822_The_Perspective_Shear-Warp_Algorithm_in_a_Virtual_Environment, fetched 2026-10-29 (unofficial)
