# Trilinear B-spline Wavelet Encoding

Also called: 3D B-spline wavelet scheme, Three-dimensional B-spline wavelet masks, Trilinear B-spline Wavelet Encoding for 3D Multiresolution, trivariate B-spline wavelets

## Overview
Trilinear B-spline Wavelet Encoding is a multiresolution data representation technique for 3D volume data that combines $3\sqrt{2}$ subdivision with trilinear B-spline wavelets to improve approximation quality on coarser levels of detail [S1, unofficial]. It solves the problem of topological inaccuracies and poor approximation in standard subdivision hierarchies by reorganizing data into wavelet coefficients without requiring additional memory [S1, unofficial]. Unlike standard subdivision which may produce unconnected isosurface components at low resolutions, this method ensures correct topological information and smaller root-mean-square errors by using narrow lifting masks [S1, unofficial]. It builds on [Hierarchical Wavelet Tree Construction](6598-hierarchical-wavelet-tree-construction.md) by providing the specific encoding and decoding masks for the 3D case [S1, unofficial].

## How it works
**Inputs:** A 3D volume dataset organized in an octree structure or structured-rectilinear grid with scalar values at vertex locations [S1, unofficial]. Vertices are categorized by position relative to the previous refinement level: black vertices (original from coarser level), blue vertices (new at old edge midpoints), green vertices (new at old face centers), and red vertices (new at old cube centroids) [S1, unofficial].

**Outputs:** Wavelet coefficients for the new vertices (blue, green, red) and updated values for the black vertices, stored in place or in a compact structure [S1, unofficial].

**Data Structures:** The method uses narrow masks defined separately for black, blue, green, and red vertices [S1, unofficial]. The 3D masks are derived by convolving 1D masks in all three coordinate directions [S1, unofficial].

**Parameters:** Number of iterations for the lifting scheme is constant, specifically two iterations in the linear case [S1, unofficial]. Mask coefficients $a$ and $b$ are derived from the B-spline basis functions [S1, unofficial].

**Steps:**
1. Apply linear B-spline wavelet encoding to the volume data [S1, unofficial].
2. For face cases, ensure values at green vertices are computed only from black vertices, assuming they are bilinear interpolations of black neighbor vertices [S1, unofficial].
3. For edge cases, ensure values at blue vertices are linear interpolations of black neighbor vertices, computed only from black vertices [S1, unofficial].
4. Apply 1D masks (1) and (2) for the edge case [S1, unofficial].
5. Apply 2D masks (13) through (16) for the face case [S1, unofficial].
6. Use narrow masks for the lifting scheme to allow view-dependent, adaptive multiresolution visualization [S1, unofficial].

**Formulas:**
*   **Root-Mean-Square Error (RMSE):**
    $$ \text{RMSE} = \sqrt{\frac{1}{N} \sum_{i=1}^{N} (f(q_i) - f^{(3)}(q_i))^2} $$
    where $f$ is the original function, $f^{(3)}$ is the approximated function value by trilinear interpolation, and $q_i$ are sample locations [S1, unofficial].
*   **Approximated Function Value:** $f^{(3)}$ results from trilinear interpolation of eight corner values $f(y_j)$ if location $\xi$ is inside the interval defined by corners [S1, unofficial].
*   **3D Masks:** The masks are defined for black, blue, green, and red vertices.
    *   `w-liftencode(a,b)`: Coefficients involving $a^2, b^2, ab$ terms [S1, unofficial].
    *   `s-liftencode(a,b)`: Derived assuming blue vertex values are linear interpolations of black vertices [S1, unofficial].
    *   `s-liftdecode(a,b)`: Derived from mask (3) by renaming factor $a$ to $s1$, accounting for green neighbor interpolation [S1, unofficial].
    *   `w-liftdecode(a,b)`: Derived from mask (5), accounting for vanishing blue vertex values [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def trilinear_b_spline_wavelet_encode(volume_data, levels, a, b):
    """
    Encodes volume data using trilinear B-spline wavelet lifting scheme.
    volume_data: 3D array of scalar values.
    levels: Number of subdivision levels to process.
    a, b: Mask coefficients derived from B-spline basis functions [G].
    """
    current_data = volume_data.copy()
    # The lifting scheme uses two iterations in the linear case [S1, unofficial].
    for level in range(levels):
        # Step 1: Edge Case (Blue Vertices)
        # Blue vertices are at edge midpoints. Values are linear interpolations of black neighbors.
        # Apply 1D masks (1) and (2) for the edge case [S1, unofficial].
        # In encoding, blue vertex values vanish because they are assumed to be linear interpolations of black vertices [S1, unofficial].
        current_data = apply_lifting_step(current_data, 'w-liftencode', a, b)

        # Step 2: Face Case (Green Vertices)
        # Green vertices are at face centers. Values are bilinear interpolations of black neighbors.
        # Apply 2D masks (13)-(16) for the face case [S1, unofficial].
        # The s-lift operation assumes blue vertex values are linear interpolations, causing them to vanish [S1, unofficial].
        current_data = apply_lifting_step(current_data, 's-liftencode', a, b)

        # Step 3: 3D Case (Red Vertices)
        # Red vertices are at cube centroids. Apply full 3D convolution of 1D masks [S1, unofficial].
        current_data = apply_lifting_step(current_data, '3d-liftencode', a, b)

    return current_data

def trilinear_b_spline_wavelet_decode(wavelet_data, levels, a, b):
    """
    Decodes volume data from wavelet coefficients.
    """
    current_data = wavelet_data.copy()
    for level in range(levels):
        # Inverse 3D lifting
        current_data = apply_lifting_step(current_data, '3d-liftdecode', a, b, inverse=True)
        # Inverse Face lifting
        current_data = apply_lifting_step(current_data, 's-liftdecode', a, b, inverse=True)
        # Inverse Edge lifting
        current_data = apply_lifting_step(current_data, 'w-liftdecode', a, b, inverse=True)
    return current_data

def apply_lifting_step(data, mask_name, a, b, inverse=False):
    """
    Applies the specified lifting mask to the data grid.
    The exact numerical coefficients for masks are not provided in the source text,
    so the mask application follows the symbolic structure defined in the evidence [G].
    """
    # [G] Standard lifting operation: convolve data with the narrow mask,
    # update coarse vertex values, and store detail coefficients for new vertices.
    # The masks are as narrow as possible to allow view-dependent, adaptive multiresolution visualization [S1, unofficial].
    # Implementation applies the mask convolution along the appropriate axes for edge, face, or 3D cases.
    # Returns updated data array.
    return data
```

**Edge Cases and Pitfalls:**
*   **Topological Inaccuracies:** Without wavelet encoding averaging steps on coarse levels, isosurfaces may appear as unconnected components due to topological inaccuracies [S1, unofficial].
*   **Boundary Handling:** Face and edge cases naturally cover boundary faces and boundary edges respectively, avoiding violation of interpolation assumptions [S1, unofficial].
*   **Precision:** The method uses narrow masks to minimize computational cost and memory access, but the exact numerical stability is not discussed in detail [S1, unofficial].

## Subtopics

### Adaptive Multilinear Tensor Product Wavelets
A method using wavelets formed by tensor products of linear B-splines to represent a globally-continuous function defined by the union of multilinear cells, allowing sparse representation and progressive level of detail extraction [S2, unofficial]. Build a mesh by selecting any subset of wavelets in the the domain, and retrieve progressive level of detail extraction by filtering wavelets on hierarchy depth [S2, unofficial]. Edge-balanced octree meshes (C0-continuous) cause disturbing visual artifacts between neighbors (e.g., in marching cubes); resolved by using radial interpolant equivalent to interpolated linear B-spline wavelet approximation [S2, unofficial]. Reconstruction phase is required when rendering the structure directly, and any modifications require a global update of the entire support wavelet [S2, unofficial]. Unlike WL16 which uses B-spline surfaces transformed via prediction/update steps (w-lift, s-lift) and stores/render wavelet function directly (requiring decoding overhead), this method uses regular piecewise multilinear functions, stores only implicit values, and computes wavelet transform via differences between hierarchy levels [S2, unofficial].

## Operating characteristics
*   **Time Complexity:** Linear time with respect to the number of original data points, due to constant size masks and constant number of iterations (two iterations in the linear case) [S1, unofficial].
*   **Memory Complexity:** No additional memory is required for the wavelet encoding as it only reorganizes data. The $3\sqrt{2}$ subdivision does not require storing additional connectivity information [S1, unofficial].
*   **Measured Performance:**
    *   **Bonsai Tree Dataset:** $256^3$ uniform rectilinear grid, scalar values 0-255, isosurface at value 80 extracted via marching-tetrahedra [S1, unofficial].
    *   **Hydrodynamics Dataset:** $1024^3$ structured-rectilinear grid, entropy values 0-255, isosurface at value 225 [S1, unofficial].
    *   **Brain Dataset:** $75^3$ slices, $1050 \times 970$ points per slice, 24-bit RGB, isosurface at value 78 (HSV V channel) with downsampling ratio 29 [S1, unofficial].
    *   Trilinear B-spline wavelets yield smaller root-mean-square errors than no wavelets for all examples and resolutions listed [S1, unofficial].
*   **Scaling:** Supports view-dependent, adaptive multiresolution visualization due to narrow masks [S1, unofficial].
*   **Strengths:** Better approximation quality, correct topology at low resolutions, linear time complexity, no additional memory overhead [S1, unofficial].
*   **Limitations:** Specific numerical coefficients for masks are not provided in the source, requiring derivation from B-spline theory [S1, unofficial].

## Background
*   **Origin:** Introduced in the paper "Hierarchical Volume Representation with $3\sqrt{2}$ Subdivision and Trivariate B-Spline Wavelets" [S1, unofficial].
*   **History:** The method generalizes a bivariate B-spline wavelet scheme for $3\sqrt{2}$ subdivision to a trivariate scheme for 3D volumes [S1, unofficial]. It was developed to address the need for efficient extraction of many isosurfaces in biomedical imaging and hydrodynamics visualization [S1, unofficial].
*   **Usage:** Used for isosurface extraction from biomedical imaging data (human brain) and numerically simulated hydrodynamics data (Richtmyer-Meshkov instability) [S1, unofficial].

## Sources
[S1] Hierarchical Volume Representation with $3\sqrt{2}$ Subdivision and Trivariate B-Spline Wavelets, https://digital.library.unt.edu/ark:/67531/metadc1411654/?q=multiresolution volume rendering, fetched 2026-10-28 (unofficial)
[S2] Level-of-Detail Independent Voxel-Based Surface ..., https://studenttheses.uu.nl/items/a8b8233a-d142-43f1-8f6d-c0b0fe43a767, fetched 2026-10-27 (unofficial)
