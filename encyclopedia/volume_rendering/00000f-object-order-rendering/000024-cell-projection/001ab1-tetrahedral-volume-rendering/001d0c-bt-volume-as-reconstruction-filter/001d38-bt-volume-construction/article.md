# BT Volume Construction

Also called: BT Volume Approximation via Least Squares

## Overview
A method to approximate an arbitrary scalar-valued volumetric function as a BT Volume using a least-squares fit [S1]. It differs from direct discretization by projecting the target function onto the polynomial basis of the BT Volume, ensuring $C^0$ continuity across tetrahedral boundaries [S1]. It serves as the primary generation technique for the data consumed by [BT Volume Isosurface Rendering](1427-bt-volume-isosurface-rendering.md) [S1].

## How it works
**Inputs:**
1. Target scalar field $f(\mathbf{x})$ defined on a domain $\Omega$ [S1].
2. A tetrahedral grid $\mathcal{T}$ (mapping $\kappa$) covering $\Omega$ [S1]. The grid voxels must overlap with the domain of the target function [S1].
3. A reconstruction filter $R(\mathbf{x})$ (optional, for the filter-based approach) [S1].

**Outputs:**
A BT Volume representation defined by vertex values and polynomial coefficients for each tetrahedron in $\mathcal{T}$ [S1].

**Data Structures:**
* **Tetrahedral Grid:** A set of tetrahedra $T_i$ [S1]. Each tetrahedron is defined by 4 vertices [S1].
* **BT Volume:** For each tetrahedron $T_i$, a set of polynomial coefficients (weights) defining the Bezier Tetrahedron [S1]. The values are stored at each vertex [S1].

**Steps:**

1. **Grid Creation:** Create a tetrahedral grid $\mathcal{T}$ that surrounds the entire target function domain [S1]. The resolution (number of tetrahedra) determines the approximation quality [S1].
    * *Note:* The specific mapping $\kappa$ determines the expressive power [S1].

2. **Sampling:** For each tetrahedron $T_i \in \mathcal{T}$:
    * Determine a set of sample points $\{\mathbf{p}_j\}$ distributed evenly in the barycentric coordinates of $T_i$ [S1].
    * Transform each barycentric point $\mathbf{p}_j$ into Euclidean space $\mathbf{x}_j$ [S1].
    * Evaluate the target function $f(\mathbf{x}_j)$ at each $\mathbf{x}_j$ [S1].

3. **Fitting (Least-Squares):**
    * Perform a least-squares fit to determine the optimal weights (vertex values) for all BTs in the volume simultaneously [S1].
    * The simultaneous fit ensures $C^0$ continuity across tetrahedral boundaries [S1].
    * *Boundary Clamping:* Artificially clamp all boundary values to zero before applying any subsequent filter convolution [S1].

4. **Filter Convolution (Optional/Advanced):**
    * If the goal is to represent the result of convolving an arbitrary scalar rectilinear volume with a reconstruction filter $R$:
        1. Generate a BT Volume approximation of the filter $R$ itself using the least-squares method above [S1].
        2. Pre-convolve the filter BT Volume with the arbitrary scalar rectilinear volume [S1].
        3. Apply the convolution equation formulation to produce a final BT Volume that equals the reconstructed data [S1].
    * This requires that the filter BT Volume shares a similar $\kappa$ function as the resulting BT Volume [S1].

**Parameters:**
* **Mapping $\kappa$:** Determines approximation quality [S1]. Key properties are: 1) Number of tetrahedra in the mapping, and 2) Spatial distribution of tetrahedra in the domain [S1]. More tetrahedra allow higher frequencies [S1].
* **Sample Density:** Number of sample points per tetrahedron [S1]. Not explicitly quantified in sources, but "distributed evenly" is specified [S1].

**Pitfalls:**
* **Continuity:** The least-squares approximation does not enforce $C^1$ continuity [S1].
* **Precision:** Graphics hardware precision issues may cause a few pixels to incorrectly miss the isosurface being rendered [S1].
* **Optimal Grid:** Determining the optimal $\kappa$ for a particular volume (balance of rendering cost vs. expressive power) is an open research problem [S1].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import numpy as np

def construct_bt_volume(target_function, domain, tetrahedral_grid, num_samples_per_tet):
    """
    Approximates target_function over domain using tetrahedral_grid.
    Returns BT Volume weights.
    """
    # 1. Grid Creation: tetrahedral_grid is provided as input.
    # Ensure grid voxels overlap with the reconstructed volume voxels.

    # 2. Sampling & 3. Fitting
    # We need to solve for weights W such that the BT Volume approximates f.
    # Let N be the total number of tetrahedra.
    # Let M be the number of sample points per tetrahedron.

    # Initialize system matrix A and vector b for least squares
    # The system solves for all vertex weights simultaneously to ensure C0 continuity.
    A = []
    b = []

    for tet in tetrahedral_grid:
        # Get 4 vertices of the tetrahedron
        v0, v1, v2, v3 = tet.vertices

        # Determine sample points in barycentric coordinates
        # "Distributed evenly" implies a standard uniform grid or quadrature
        # in the reference tetrahedron.
        sample_points_bary = _generate_even_barycentric_samples(num_samples_per_tet)

        for bary in sample_points_bary:
            # Transform to Euclidean space
            x = _barycentric_to_euclidean(bary, v0, v1, v2, v3)

            # Evaluate target function
            f_val = target_function(x)

            # Compute basis function values for this point
            # For a BT Volume, the value at x is a linear combination of vertex weights
            # weighted by the Bernstein polynomials (or equivalent basis) of the tet.
            basis_values = _compute_bt_basis(bary)

            # Add row to A and value to b
            # A * W = b
            A.append(basis_values)
            b.append(f_val)

    # Solve least squares: min ||A*W - b||^2
    # W is the vector of all vertex weights for the entire grid
    # Using normal equations for standard least-squares solve [G]
    A_np = np.array(A)
    b_np = np.array(b)
    W = np.linalg.solve(A_np.T @ A_np, A_np.T @ b_np)

    # 4. Boundary Clamping (if preparing for filter convolution)
    # Artificially clamp all boundary values to zero.
    # Identify boundary vertices of the grid.
    boundary_vertices = _get_boundary_vertices(tetrahedral_grid)
    for v in boundary_vertices:
        W[v] = 0.0

    return W

def construct_filter_bt_volume(filter_function, domain, tetrahedral_grid, num_samples_per_tet):
    """
    Generates a BT Volume representation of a reconstruction filter.
    """
    # Use the same least-squares technique as above
    weights = construct_bt_volume(filter_function, domain, tetrahedral_grid, num_samples_per_tet)
    return weights

def pre_convolve_filter_bt(filter_bt_weights, scalar_volume, grid):
    """
    Pre-convolves the filter BT Volume with an arbitrary scalar rectilinear volume.
    """
    # Apply the convolution equation formulation to produce a final BT Volume.
    # This results in a BT Volume that equals the reconstructed data.
    # Implementation details of the specific convolution equation are not provided
    # in the source text beyond the high-level description. [G]
    # This step likely involves computing the convolution of the scalar volume
    # with the basis functions of the filter BT, then projecting back to the BT basis.
    # final_bt_weights = _apply_convolution_equation(filter_bt_weights, scalar_volume, grid)
    # return final_bt_weights
    pass

def _generate_even_barycentric_samples(num_samples):
    """Generates sample points uniformly distributed in barycentric coordinates."""
    # Standard barycentric grid generation [G]
    samples = []
    for i in range(num_samples + 1):
        for j in range(num_samples + 1 - i):
            k = num_samples - i - j
            samples.append(np.array([i, j, k]) / num_samples)
    return samples

def _barycentric_to_euclidean(bary, v0, v1, v2, v3):
    """Transforms barycentric coordinates to Euclidean space."""
    return bary[0] * v0 + bary[1] * v1 + bary[2] * v2 + bary[3] * v3

def _compute_bt_basis(bary):
    """Computes the polynomial basis values for a point in barycentric coordinates."""
    # Exact polynomial degree is not specified in the sources. [G]
    # Returns a vector of basis function evaluations for the given barycentric coordinates.
    return np.ones(4)  # Placeholder for basis evaluation; degree unspecified [G]

def _get_boundary_vertices(tetrahedral_grid):
    """Identifies vertices that belong to only one tetrahedron."""
    # Standard mesh boundary detection [G]
    vertex_counts = {}
    for tet in tetrahedral_grid:
        for v in tet.vertices:
            vertex_counts[v] = vertex_counts.get(v, 0) + 1
    return [v for v, count in vertex_counts.items() if count == 1]

def _apply_convolution_equation(filter_bt_weights, scalar_volume, grid):
    """Applies the convolution equation formulation to produce the final BT Volume."""
    # The exact convolution equation formulation is not provided in the sources. [G]
    # Returns the convolved BT Volume weights.
    return filter_bt_weights  # Placeholder for convolution logic [G]
```

## Operating characteristics
* **Complexity:** Not explicitly given in the sources. The least-squares fit involves solving a linear system where the size depends on the number of tetrahedra and sample points [S1].
* **Performance:** The method is a pre-processing step. The resulting BT Volume allows for efficient rendering via local ray-intersection equations on the graphics card [S1].
* **Strengths:**
    * Provides a continuous $C^0$ representation of arbitrary scalar fields [S1].
    * Allows the use of BT Volumes as reconstruction filters, enabling efficient convolution [S1].
* **Limitations:**
    * Does not enforce $C^1$ continuity [S1].
    * Graphics hardware precision issues can cause rendering artifacts (missed isosurfaces) [S1].
    * Determining the optimal tetrahedral grid ($\kappa$) is an open problem [S1].

## Background
* **Origin:** Introduced in "Interactive volume isosurface rendering using BT volumes" (2008) [S1].
* **Development:** The method was developed as part of a broader framework for rendering BT Volumes, which are polynomial solids defined on tetrahedral grids with special regularity requirements [S1].
* **Usage:** Used to generate BT Volume data from arbitrary scalar fields or to create BT Volume representations of reconstruction filters for convolution [S1].

## Sources
[S1] Interactive volume isosurface rendering using BT volumes, https://doi.org/10.1145/1342250.1342257, fetched 2026-10-30
