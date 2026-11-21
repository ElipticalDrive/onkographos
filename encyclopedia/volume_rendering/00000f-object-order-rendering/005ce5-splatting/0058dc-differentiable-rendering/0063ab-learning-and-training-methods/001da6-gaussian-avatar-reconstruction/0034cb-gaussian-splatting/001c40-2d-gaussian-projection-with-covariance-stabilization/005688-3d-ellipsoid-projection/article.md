# 3D Ellipsoid Projection

Also called: Gaussian Selection Method, Sphere Approximation Projection

## Overview
3D Ellipsoid Projection is a Gaussian selection method used to identify candidate 3D Gaussians for rendering specific image slices in volume rendering pipelines. It approximates the spatial extent of a 3D Gaussian ellipsoid as a sphere to compute a uniform 2D bounding box, which is then projected orthogonally onto every image slice. Unlike sibling methods that adapt bounding boxes per slice or use complex covariance corrections, this approach prioritizes simplicity by treating the Gaussian's range as isotropic, though this leads to redundant computation for off-center slices.

## How it works
**Inputs:**
*   A 3D Gaussian primitive defined by its mean position $\mathbf{\mu}$ and covariance matrix $\Sigma$.
*   A set of image slices (depth planes) along the viewing direction.

**Outputs:**
*   A set of candidate Gaussians for each slice, determined by whether the projected bounding box intersects the slice's region of interest.

**Data Structures:**
*   **Covariance Matrix $\Sigma$**: A $3 \times 3$ symmetric positive-definite matrix representing the Gaussian's shape and orientation.
*   **Eigenvalues $\lambda_1, \lambda_2, \lambda_3$**: The scalar values obtained from the spectral decomposition of $\Sigma$.
*   **Bounding Box**: A cube defined by its center $\mathbf{\mu}$ and edge length $2 r_{max}$.

**Steps:**
1.  **Compute Eigenvalues**: Calculate the eigenvalues $\lambda_1, \lambda_2, \lambda_3$ of the covariance matrix $\Sigma$.
2.  **Determine Maximum Axis Length**: Identify the largest eigenvalue $\lambda_{max} = \max(\lambda_1, \lambda_2, \lambda_3)$.
3.  **Calculate Approximation Radius**: Compute the radius $r_{max}$ of the approximating sphere using the formula:
    $$ r_{max} = 3 \cdot \sqrt{\lambda_{max}} $$
    The factor of 3 corresponds to the $3\sigma$ confidence interval, encompassing approximately 99.7% of the Gaussian's probability mass.
4.  **Define Bounding Cube**: Construct a cube centered at the Gaussian's mean $\mathbf{\mu}$ with edge length $2 r_{max}$.
5.  **Project to Slices**: Orthogonally project this cube onto each image slice. Because the projection is orthogonal and the shape is a cube aligned with the principal axes (or simply treated as a sphere's bounding box), the resulting 2D bounding box is identical for all slices.
6.  **Selection**: Include the Gaussian in the candidate set for a slice if the slice's pixel coordinates fall within the projected 2D bounding box.

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def compute_3d_ellipsoid_projection(gaussian_mean, covariance_matrix, slices):
    """
    gaussian_mean: 3D vector (x, y, z)
    covariance_matrix: 3x3 matrix
    slices: list of slice objects or depth values
    
    Returns: list of (slice_index, bounding_box_2d) for all slices
    """
    # Step 1 & 2: Compute eigenvalues and find max
    eigenvalues = compute_eigenvalues(covariance_matrix)
    lambda_max = max(eigenvalues)
    
    # Step 3: Calculate radius
    r_max = 3.0 * math.sqrt(lambda_max)
    
    # Step 4: Define cube dimensions
    edge_length = 2.0 * r_max
    half_edge = r_max
    
    # Step 5: Project to all slices
    results = []
    for i, slice_obj in enumerate(slices):
        # The bounding box is identical for all slices
        # Assuming slice_obj has a method to check intersection or we return the box
        # The box is centered at gaussian_mean projected onto the slice plane
        # For orthogonal projection along Z (assuming standard view), 
        # the 2D center is (mean_x, mean_y)
        
        center_2d = (gaussian_mean.x, gaussian_mean.y)
        bounding_box_2d = {
            "center": center_2d,
            "half_width": half_edge,
            "half_height": half_edge
        }
        results.append((i, bounding_box_2d))
        
    return results

def compute_eigenvalues(matrix):
    """
    Computes eigenvalues of a 3x3 symmetric matrix.
    In a real implementation, use a standard linear algebra library routine.
    """
    # Placeholder for actual eigenvalue decomposition logic
    # [G] Standard algorithm: Solve det(A - lambda*I) = 0
    # For a 3x3 symmetric matrix, this can be done via Jacobi eigenvalue algorithm
    # or characteristic polynomial roots.
    pass
```

**Parameters:**
*   **Confidence Factor**: 3 (representing $3\sigma$). This is a standard statistical choice for bounding boxes in Gaussian rendering.
*   **Epsilon**: Not explicitly defined in the source for this specific method, but numerical stability in eigenvalue computation should be considered [G].

**Edge Cases and Pitfalls:**
*   **Redundant Computation**: Because the same cubic bounding box is projected onto every slice, slices far from the Gaussian's center may still be processed if they fall within the large projected box, leading to unnecessary rasterization work [S1, unofficial].
*   **Overestimation**: The spherical approximation ignores the orientation of the ellipsoid. If the Gaussian is highly anisotropic (e.g., a flat sheet), the sphere will include large volumes of empty space, further exacerbating the redundant computation issue [S1, unofficial].

## Subtopics

### Conditional Splatting
A Gaussian selection method that uses a factorized Gaussian structure to compute the range of each Gaussian on individual slices, adapting the bounding box for each slice based on depth. Unlike 3D Ellipsoid Projection, which projects a static cubic bounding box onto every slice, Conditional Splatting computes an adapted bounding box for each slice. It models the Gaussian contribution as $p_i(u, v, t) = p_i(u, v|t)p_i(t)$, where $p_i(u, v|t)$ is a conditional 2D Gaussian describing the lateral spatial distribution given slice location $t$, and $p_i(t)$ is a 1D Gaussian modeling uncertainty along the depth axis. The conditional mean $\mu_{u,v|t}$ shifts with slice depth $t$, and a distance-dependent scaling factor reduces the extent of the 2D splat as the distance between the 3D Gaussian center and the slice increases. This leads to more efficient sampling by avoiding the processing of overly large, static regions [S1, unofficial].

## Operating characteristics
*   **Time Complexity**: $O(N \cdot S)$, where $N$ is the number of Gaussians and $S$ is the number of slices. Each Gaussian is evaluated against every slice.
*   **Memory Complexity**: Low, as it only requires storing the bounding box parameters per slice.
*   **Performance**: The method is computationally simple to implement but inefficient in scenes with many Gaussians or deep volumes, as it processes redundant pixels on slices distant from the Gaussian's center [S1, unofficial].
*   **Strengths**: Simple to implement; robust to orientation changes as it relies on the maximum axis length.
*   **Limitations**: High false-positive rate for candidate selection; significant redundant computation due to identical bounding boxes across all slices [S1, unofficial].
*   **When to use**: When implementation simplicity is prioritized over rendering speed, or in shallow volumes where the redundancy is minimal.
*   **When not to use**: In deep volumetric scenes or with highly anisotropic Gaussians, where adaptive methods like Conditional Splatting offer better efficiency [S1, unofficial].

## Background
The 3D Ellipsoid Projection method was introduced in the context of internal scene reconstruction and segmentation using factorized 3D Gaussian Splatting. It serves as a baseline Gaussian selection strategy to identify candidate Gaussians for each pixel/slice before differential rasterization. The method was compared against Conditional Splatting in simulation analyses involving 3D volumes of size $20 \times 20 \times 20$ [S1, unofficial].

## Sources
[S1] InnerGS: Internal Scenes Reconstruction and Segmentation via Factorized 3D Gaussian Splatting, https://doi.org/10.48550/arxiv.2508.13287, fetched 2026-10-26 (unofficial)
