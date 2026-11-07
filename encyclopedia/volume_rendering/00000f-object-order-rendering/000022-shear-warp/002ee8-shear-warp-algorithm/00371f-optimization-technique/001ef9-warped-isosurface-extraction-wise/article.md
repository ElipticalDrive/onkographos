# Warped Isosurface Extraction (WISE)
## Overview
Warped Isosurface Extraction (WISE) is a method to accelerate visibility tests in volume rendering by using shear-warp factorization to replace expensive perspective projections with cheaper orthographic-like projections on a warped plane [S1, book]. It sets itself apart from standard perspective projection by eliminating the nonlinearity of the view transformation for visibility culling, allowing precomputed per-slice coefficients for grid points and reducing the computational cost of vertex projection significantly [S1, book].

## How it works
WISE accelerates the visibility testing phase of isosurface extraction by projecting geometry onto a warped plane rather than the final image plane [S1, book]. The core insight is that visibility is preserved between the image plane and the warped projection plane, meaning if a primitive is occluded on the warped plane, it is occluded on the final image [S1, book].

### Inputs and Outputs
*   **Inputs**: Dataset coordinate system $[X, Y, Z]$ with scaling vector $[sx, sy, sz]$ [S1, book]. Eye position $[eyex, eyey, eyez]$ transformed into the dataset coordinate system [S1, book]. Geometry to test, consisting of either grid points $p(px, py, pz)$ or isosurface triangle vertices [S1, book].
*   **Outputs**: Projected 2D coordinates $(x, y)$ on the warped plane $Z=0$ [S1, book]. Visibility status indicating whether the geometry is visible or occluded [S1, book].

### Data Structures
*   **Precomputed Coefficients**: Four floating-point arrays indexed by Z-slice $Z$, storing `scalex[Z]`, `scaley[Z]`, `shearx[Z]`, and `sheary[Z]` [S1, book].
*   **Geometry**: Triangle vertices defined by 3D coordinates $(px, py, pz)$ [S1, book].

### Steps
1.  **Factor the Viewing Transformation**: Decompose the view into a shear followed by a warp transformation [S1, book].
2.  **Precompute Coefficients**: For the current viewpoint, calculate shear and scale factors for each Z-slice [S1, book].
3.  **Project Geometry**: Map grid points using the precomputed coefficients, and map isosurface triangle vertices using the specific orthographic projection formula derived from the warped space [S1, book].
4.  **Perform Visibility Tests**: Conduct occlusion tests on the 2D warped plane, as visibility is preserved between the warped plane and the image plane [S1, book].
5.  **Warp (Optional for Visibility)**: If the final image is required, apply the 2D warp transformation to the intermediate image [S1, book].

### Formulas
The warped projection plane is assumed to be $Z=0$ [S1, book].

**Precomputation (per Z-slice):**
Let $Z$ be the slice index.
$$ s = \frac{Z \cdot sz}{Z \cdot sz - eyez} $$
$$ scalex[Z] = (1 - s) \cdot sx $$
$$ scaley[Z] = (1 - s) \cdot sy $$
$$ shearx[Z] = s \cdot eyex $$
$$ sheary[Z] = s \cdot eyey $$

**Projection for Grid Points:**
For a point $p(px, py, pz)$:
$$ x = px \cdot scalex[pz] + shearx[pz] $$
$$ y = py \cdot scaley[pz] + sheary[pz] $$
Cost: 2 multiplications and 2 additions per vertex [S1, book].

**Projection for Isosurface Triangle Vertices:**
For a vertex $P(px, py, pz)$:
$$ s = \frac{pz}{pz - eyez} $$
$$ x = px + s \cdot (eyex - px) $$
$$ y = py + s \cdot (eyey - py) $$
Cost: 2 multiplications, 5 additions, and 1 division per vertex [S1, book].

### Parameters
*   Warped projection plane: $Z=0$ [S1, book].
*   Typical values for dataset scaling and eye position: Not given in the sources [S1, book].

### Pseudocode
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def precompute_wise_coefficients(eye_x, eye_y, eye_z, scale_x, scale_y, scale_z, num_slices):
    """
    Precompute shear and scale coefficients for each Z-slice.
    Assumes warped projection plane is Z=0.
    """
    scalex = [0.0] * num_slices
    scaley = [0.0] * num_slices
    shearx = [0.0] * num_slices
    sheary = [0.0] * num_slices

    for z in range(num_slices):
        denominator = z * scale_z - eye_z
        if denominator == 0:
            # Guard for degenerate case: eye is exactly on the slice plane [G]
            s = 0.0 
        else:
            s = (z * scale_z) / denominator
        
        scalex[z] = (1.0 - s) * scale_x
        scaley[z] = (1.0 - s) * scale_y
        shearx[z] = s * eye_x
        sheary[z] = s * eye_y

    return scalex, scaley, shearx, sheary

def project_grid_point(px, py, pz, scalex, scaley, shearx, sheary):
    """
    Project a grid point using precomputed coefficients.
    Returns (x, y) on the warped plane.
    """
    # pz is used directly as an integer slice index per the source formulation [S1, book]
    x = px * scalex[pz] + shearx[pz]
    y = py * scaley[pz] + sheary[pz]
    return x, y

def project_triangle_vertex(px, py, pz, eye_x, eye_y, eye_z):
    """
    Project an isosurface triangle vertex onto the warped plane.
    Returns (x, y) on the warped plane.
    """
    denominator = pz - eye_z
    if denominator == 0:
        # Guard: vertex at eye position yields undefined projection [G]
        return None 
    
    s = pz / denominator
    x = px + s * (eye_x - px)
    y = py + s * (eye_y - py)
    return x, y
```

### Edge Cases and Pitfalls
*   **Division by Zero**: If $pz = eyez$ for a triangle vertex, the projection is undefined [G]. A guard should check for this condition and either discard the primitive or handle it as a special case [G].
*   **Precision**: The precomputation for grid points relies on $Z$ being an integer index [G]. Ensure $Z$ indices map correctly to the precomputed arrays [G].
*   **Visibility Preservation**: The method assumes that occlusion on the warped plane implies occlusion on the image plane [S1, book].

## Operating characteristics
*   **Time Complexity**: Grid point projection requires $O(1)$ time per vertex, specifically 2 multiplications and 2 additions [S1, book]. Triangle vertex projection requires $O(1)$ time per vertex, specifically 2 multiplications, 5 additions, and 1 division [S1, book]. Standard perspective projection requires 16 multiplications, 12 additions, and 2 divisions per vertex [S1, book].
*   **Memory Complexity**: The method requires $O(N)$ memory where $N$ is the number of Z-slices, to store the precomputed coefficients [S1, book].
*   **Strengths**: The method significantly reduces arithmetic operations for visibility testing [S1, book]. It enables precomputation of transformation factors per slice, decoupling them from individual vertex processing for grid points [S1, book]. It preserves visibility, allowing culling on the warped plane [S1, book].
*   **Limitations**: The method requires precomputation for each new viewpoint [S1, book]. Triangle vertices still require a division, though fewer than perspective projection [S1, book]. Performance impact on modern GPUs with hardware perspective correction is not detailed in the sources [S1, book].

## Background
WISE is based on the shear-warp factorization of the viewing transformation, originally presented by Lacroute in 1994 [S1, book]. The method was developed to accelerate visibility tests in large-scale isosurface extraction, where the cost of perspective projection for every vertex of meta-cells and triangles is prohibitive [S1, book]. It is used in volume rendering and isosurface extraction for large image datasets [S1, book].

## Sources
[S1] Fast Isosurface Extraction Methods for Large Image Data sets, https://publications.sci.utah.edu/publications/livnat08/Large-scale-isosurface08.pdf, fetched 2026-10-28 (book)
