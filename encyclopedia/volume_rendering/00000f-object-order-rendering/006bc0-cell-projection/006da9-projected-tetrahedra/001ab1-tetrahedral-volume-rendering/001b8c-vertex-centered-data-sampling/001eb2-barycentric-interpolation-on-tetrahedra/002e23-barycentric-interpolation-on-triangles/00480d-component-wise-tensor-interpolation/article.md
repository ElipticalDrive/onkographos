# Component-wise Tensor Interpolation

Also called: Linear combination of edge tensors, Trilinear interpolation on triangle plane

## Overview
Component-wise tensor interpolation is a method for reconstructing second-order tensor fields at arbitrary points within a discretized domain by treating the tensor as a linear combination of surrounding edge tensors [S1, unofficial]. It solves the problem of orientation ambiguities and non-unit vector issues that arise when interpolating derived quantities like eigenvectors directly [S1, unofficial]. Unlike interpolating raw MRI data, which is computationally enormous, or interpolating eigenvectors, which suffers from orientation discontinuities, this method preserves tensor symmetry and reduces computational effort by operating on tensor components directly [S1, unofficial].

## How it works
**Inputs:**
- A point $p$ located inside a geometric element (triangle or volume element).
- Barycentric coordinates $\beta_k$ for point $p$ relative to the element's vertices or edge centers.
- Second-order tensors $T^k$ stored at the edges (or vertices, depending on the specific discretization context, though the definition specifies edge tensors) of the element.

**Outputs:**
- The interpolated second-order tensor $T$ at point $p$.

**Data Structures:**
- Tensors are represented as $3 \times 3$ matrices (or symmetric $3 \times 3$ matrices). Each tensor $T^k$ has components $T^k_{ij}$ where $i,j \in \{1,2,3\}$.
- Barycentric coordinates $\beta_k$ are scalars in the range $[0, 1]$.

**Steps:**
1. Calculate the barycentric coordinates $\beta_k$ for the point $p$ inside the triangle or volume element [S1, unofficial].
2. For each component index pair $(i, j)$, compute the weighted sum of the corresponding tensor components from the surrounding edges:
   $$ \text{numerator}_{ij} = \sum_{0 < k < 3} \beta_k T_{ij}^k $$
3. Compute the sum of the barycentric coordinates:
   $$ \text{denominator} = \sum_{0 < k < 3} \beta_k $$
4. Divide the weighted sum by the sum of barycentric coordinates to get the interpolated component:
   $$ T_{ij} = \frac{\text{numerator}_{ij}}{\text{denominator}} $$

**Formula:**
The interpolated tensor component $T_{ij}$ is defined as [S1, unofficial]:
$$ T_{ij} = \frac{\sum_{0 < k < 3} \beta_k T_{ij}^k}{\sum_{0 < k < 3} \beta_k} $$

**Parameters:**
- $\beta_k$: Barycentric coordinates for point $p$, values in range $[0, 1]$ [S1, unofficial].
- $T^k_{ij}$: The components of the second-order tensors at the triangle edges ($k=1,2,3$) [S1, unofficial].

**Pitfalls:**
- Naive interpolation of eigenvectors with different orientations causes rotation of vectors and potential zero vectors due to numerical instabilities [S1, unofficial]. This method avoids that by interpolating tensor components directly [S1, unofficial].
- The sum of barycentric coordinates in the denominator must be non-zero. If $p$ coincides with a vertex or edge where coordinates might degenerate, special handling is required [G].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def component_wise_tensor_interpolation(p, barycentric_coords, edge_tensors):
    """
    Interpolates a second-order tensor at point p using component-wise interpolation.
    
    Args:
        p: Point inside the triangle/volume (not explicitly used in formula, 
           barycentric_coords are sufficient).
        barycentric_coords: List of barycentric coordinates [beta_1, beta_2, beta_3].
        edge_tensors: List of 3x3 matrices (or symmetric 3x3 matrices) T^1, T^2, T^3.
        
    Returns:
        Interpolated 3x3 tensor matrix T.
    """
    # Sum of barycentric coordinates
    sum_beta = sum(barycentric_coords)
    
    # Check for degenerate case where sum_beta is zero
    if sum_beta == 0:
        raise ValueError("Sum of barycentric coordinates is zero. Point p is likely undefined or at a singularity.")
        
    # Initialize result tensor with zeros
    T = [[0.0 for _ in range(3)] for _ in range(3)]
    
    # Interpolate each component
    for i in range(3):
        for j in range(3):
            weighted_sum = 0.0
            for k in range(3):
                beta_k = barycentric_coords[k]
                T_k_ij = edge_tensors[k][i][j]
                weighted_sum += beta_k * T_k_ij
            T[i][j] = weighted_sum / sum_beta
            
    return T
```

## Subtopics

### Linear Eigenvector Interpolation with Edge Labels
An alternative method for tensor fields on triangular meshes that ensures consistent eigenvector orientation across triangle boundaries by using edge labels encoding relative orientation, allowing simple vector interpolation inside triangles [S2, book]. It computes edge labels once to encode the relative orientation of eigenvectors in adjacent vertices, then performs linear vector interpolation inside triangles [S2, book]. A pitfall is that component-based interpolation leads to higher dimensionality degenerate entities, whereas eigenvector-based interpolation handles degenerate lines/triangles frequently by treating polyline vertices as degenerate points with constant eigenvector fields in adjacent triangles [S2, book]. This method minimizes the number of eigenvector computations and allows exact integration of tensorlines, making it suitable for performance-critical applications like topological skeleton extraction [S2, book].

## Operating characteristics
- **Time Complexity:** Linear with respect to the number of edges/vertices defining the element (constant for a triangle). Component-wise interpolation is computationally cheaper than computing eigenvectors/eigenvalues at every point [S1, unofficial].
- **Memory Complexity:** Requires storing tensor components at edges/vertices. No additional storage for eigenvectors or orientation labels if component-wise interpolation is used [S1, unofficial].
- **Strengths:** Preserves symmetry of tensors; avoids orientation ambiguities; computationally efficient compared to raw data interpolation or eigenvector-based methods [S1, unofficial].
- **Limitations:** May not capture topological features as accurately as eigenvector-based methods in the presence of degenerate points or lines, as component-based interpolation can lead to higher dimensionality degenerate entities [S2, book].
- **When to use:** Real-time visualization of tensor fields where performance is critical and exact topological accuracy is less important than smooth visual appearance [S1, unofficial].
- **When not to use:** When exact integration of tensorlines or topological skeleton extraction is required, especially in the presence of degenerate points, lines, or triangles [S2, book].

## Background
The method was introduced in the context of image-space tensor field visualization, specifically for diffusion tensor imaging (DTI) data [S1, unofficial]. It addresses the challenges of visualizing second-order tensor fields, which are commonly acquired from MRI scanners [S1, unofficial]. The approach was proposed as a more efficient alternative to interpolating raw MRI data or derived quantities like eigenvectors, which suffer from orientation problems [S1, unofficial]. The definition of trilinear interpolation on a triangle plane for tensors is formally presented in Definition 2.17 of the source [S1, unofficial].

## Sources
[S1] Image-Space Tensor Field Visualization Using a LIC-like Method, https://doi.org/10.1007/978-3-642-21608-4_11, fetched 2026-10-27 (unofficial)
[S2] DFU, Volume 2, Scientific Visualization: Interactions, Features, Metaphors, Complete Volume, https://doi.org/10.4230/dfu.vol2.sciviz.2011, fetched 2026-10-26 (book)
