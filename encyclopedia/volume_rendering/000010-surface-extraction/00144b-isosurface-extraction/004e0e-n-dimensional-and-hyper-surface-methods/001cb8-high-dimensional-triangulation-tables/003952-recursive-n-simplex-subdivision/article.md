# Recursive n-simplex subdivision

Also called: Splitting n-dimensional cells into n-simplexes

## Overview
Recursive n-simplex subdivision is an isosurface extraction method for scalar fields in $n$ dimensions ($n > 3$) that decomposes hypercubic cells into $n$-simplexes to facilitate triangulation [S1, unofficial]. It addresses the complexity of generating surfaces in high-dimensional spaces where standard marching cubes lookup tables become prohibitively large [S1, unofficial]. Unlike [High-Dimensional Triangulation Tables](1336-high-dimensional-triangulation-tables.md), which pre-computes triangulation configurations for hypercubes, this method recursively splits cells into simplexes, relying on a centroid-based division that is noted as suboptimal compared to lookup-table-driven approaches [S1, unofficial].

## How it works
**Inputs:** An $n$-dimensional scalar field defined on a regular grid.
**Outputs:** A set of $(n-1)$-simplexes representing the isosurface.

**Data Structures:**
The method operates on $n$-dimensional hypercubic cells. Each cell is recursively subdivided into $n$-simplexes. The specific geometric layout of the resulting simplexes is determined by the centroid division technique.

**Steps:**
1.  **Cell Decomposition:** For each $n$-dimensional hypercubic cell in the grid, apply a recursive technique to split the cell into $n$-simplexes [S1, unofficial].
2.  **Surface Computation:** Compute the isosurface intersections within each $n$-simplex. The resulting surfaces exist in $n$ dimensions (specifically 4D in the cited implementations) [S1, unofficial].
3.  **Dimensionality Reduction:** Display the resulting high-dimensional surfaces using one of two techniques:
    *   Stereographic projection [S1, unofficial].
    *   Slicing to reduce dimensionality [S1, unofficial].

**Parameters and Pitfalls:**
*   **Division Technique:** The method uses a centroid division technique to break cells into simplexes [S1, unofficial].
*   **Optimization:** This centroid division is suboptimal [S1, unofficial]. It can be improved by using lookup tables similar to those used in marching cubes [S1, unofficial].
*   **Complexity:** An unavoidable consequence of increasing dimensionality is an increase in the complexity of storage and computation [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def recursive_n_simplex_subdivision(field, n, isovalue):
    """
    Extract isosurface from an n-dimensional scalar field.
    
    Args:
        field: n-dimensional array of scalar values.
        n: Dimensionality of the field (e.g., 4).
        isovalue: The threshold value for the isosurface.
        
    Returns:
        list of (n-1)-simplexes representing the surface.
    """
    surfaces = []
    
    # Iterate over all n-dimensional hypercubic cells in the grid
    for cell in get_n_cubes(field):
        # Step 1: Split the n-dimensional cell into n-simplexes
        # Using the centroid division technique
        simplexes = split_cell_into_n_simplexes_centroid(cell)
        
        # Step 2: Compute surfaces in n dimensions (e.g., 4D)
        # For each simplex, determine intersection with isovalue
        for simplex in simplexes:
            surface_patch = compute_isosurface_in_simplex(simplex, isovalue)
            if surface_patch is not None:
                surfaces.append(surface_patch)
                
    return surfaces

def split_cell_into_n_simplexes_centroid(cell):
    """
    Recursively split an n-dimensional hypercube into n-simplexes
    using the centroid division technique.
    
    Note: This is suboptimal. Lookup tables similar to marching cubes
    can improve this [S1, unofficial].
    """
    # Implementation details for centroid division are not explicitly
    # provided in the source, but the method is identified as recursive.
    # [G] Standard centroid decomposition involves connecting the
    # centroid of the hypercube to its vertices and recursively
    # decomposing sub-cells.
    return decompose_hypercube(cell)

def decompose_hypercube(cell):
    """
    Placeholder for the specific recursive splitting logic.
    The source notes the technique is recursive but does not provide
    the exact geometric decomposition algorithm.
    """
    # [G] In n-dimensions, a common recursive split involves
    # dividing the hypercube into n! simplexes or using a
    # specific triangulation scheme. The source mentions
    # "recursive technique" and "centroid division".
    pass
```

## Subtopics

### [Wavelet Representation of Contour Sets](21995-wavelet-representation-of-contour-sets.md)
Represents scalar field contours as a hierarchy of meshes using subdivision surfaces and wavelets. Unlike standard wavelet schemes that quantize field values on a regular grid, this approach compresses a parametrization of the scalar field induced by its contours, preserving prescribed contour topology at every resolution level.

## Operating characteristics
*   **Complexity:** Increasing the dimensionality of the input field leads to an increase in the complexity of its storage and computation [S1, unofficial].
*   **Performance:** The centroid division technique is suboptimal [S1, unofficial].
*   **Strengths:** Allows for the computation of surfaces in four dimensions and beyond [S1, unofficial].
*   **Limitations:** The method produces surfaces in 4D which require stereographic projection or slicing for display [S1, unofficial]. The subdivision technique is noted as suboptimal compared to lookup-table methods [S1, unofficial].
*   **Usage:** Used to compute swept volumes of time-varying data from electromagnetic field simulations, allowing display as animations [S1, unofficial].

## Background
An early example of computing surfaces in more than three dimensions was considered by Weigle and Banks [74] using a recursive technique to split n-dimensional cells into n-simplexes [S1, unofficial]. The technique was used in [75] to compute swept volumes of time-varying data generated from electromagnetic field simulations [S1, unofficial]. Extending marching cubes into the fourth dimension and beyond was further explored by Bhaniramka et al. [76], who found that the lookup table required 222 separate configurations after removing symmetrical configurations [S1, unofficial].

## Sources
[S1] Chapter Topological Visualisation Techniques for Volume Multifield Data, https://archive.org/details/oapen-20.500.12657-49310, fetched 2026-10-27 (unofficial)
