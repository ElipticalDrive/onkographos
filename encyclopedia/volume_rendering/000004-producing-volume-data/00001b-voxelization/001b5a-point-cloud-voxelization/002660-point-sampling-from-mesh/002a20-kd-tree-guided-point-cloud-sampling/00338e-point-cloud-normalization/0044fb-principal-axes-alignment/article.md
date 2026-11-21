# Principal Axes Alignment


## Overview
Principal Axes Alignment computes a 4x4 SE(3) transform that aligns a point cloud to its principal axes of covariance while centering it at the median [S1, unofficial]. It serves as the final normalization step in the point cloud preprocessing pipeline, ensuring that subsequent geometric operations (such as coarse geometry prediction or differentiable rendering optimization) operate on a standardized coordinate frame [S1, unofficial]. It differs from sibling methods by explicitly decoupling scale initialization from rotational initialization and by using the median for centering rather than the mean, which provides robustness against outliers [S1, unofficial].

## How it works
**Inputs and Outputs**
- Input: A point cloud represented as an $N \times 3$ matrix of coordinates $P$, where each row is a point $(x, y, z)$.
- Output: A $4 \times 4$ homogeneous transformation matrix $T$ (SE(3)) that maps the original point cloud to the aligned space.

**Data Structures**
- $P$: $N \times 3$ float array.
- $C$: $1 \times 3$ centroid vector.
- $\Sigma$: $3 \times 3$ covariance matrix.
- $E$: $3 \times 3$ matrix of eigenvectors (columns).
- $\Lambda$: $3 \times 3$ diagonal matrix of eigenvalues (or $1 \times 3$ vector).
- $T$: $4 \times 4$ identity matrix initialized, then modified.

**Steps**
1. **Compute Centroid**: Calculate the median of the point cloud along the point axis.
   $$ C = \text{median}(P, \text{axis}=0) $$
   [S1, unofficial]

2. **Translate to Centroid**: Subtract the centroid from all points to center the cloud at the origin.
   $$ P' = P - C $$
   [S1, unofficial]

3. **Compute Covariance Matrix**: Calculate the covariance matrix of the centered points.
   $$ \Sigma = \text{cov}(P', \text{rowvar}=False) $$
   [S1, unofficial]

4. **Eigen Decomposition**: Compute eigenvalues and eigenvectors of $\Sigma$.
   $$ \Lambda, E = \text{eigh}(\Sigma) $$
   [S1, unofficial]
   Note: `eigh` returns eigenvalues in ascending order.

5. **Sort Eigenvectors**: Sort the eigenvectors corresponding to eigenvalues in descending order. The first column of the resulting rotation matrix corresponds to the principal axis with the largest variance, and the last column (z-axis) corresponds to the smallest eigenvalue.
   $$ \text{sort\_indices} = \text{argsort}(\Lambda)[::-1] $$
   $$ E_{sorted} = E[:, \text{sort\_indices}] $$
   [S1, unofficial]

6. **Ensure Right-Handed System**: Check the determinant of the rotation matrix. If $\det(E_{sorted}) < 0$, flip the sign of the first eigenvector to ensure a proper rotation (right-handed coordinate system).
   $$ \text{if } \det(E_{sorted}) < 0: \quad E_{sorted}[:, 0] *= -1 $$
   [S1, unofficial]

7. **Construct Transform**: Build the $4 \times 4$ SE(3) transform. The rotation part is $E_{sorted}^T$. The translation part ensures that the original centroid $C$ is mapped to the origin in the new frame (or equivalently, the origin in the new frame maps to $C$ in the original frame, depending on application; here, the transform aligns the cloud, so we apply rotation then translation to align the cloud to axes, but the standard alignment transform usually centers the cloud. The source specifies `transform[:3, 3] = -rotation_matrix @ centroid`).
   $$ R = E_{sorted}^T $$
   $$ T = \begin{bmatrix} R & -R C \\ 0 & 1 \end{bmatrix} $$
   [S1, unofficial]

**Pseudocode**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import numpy as np

def principal_axes_alignment(point_cloud: np.ndarray) -> np.ndarray:
    """
    Aligns a point cloud to its principal axes and centers it at the median.
    
    Args:
        point_cloud: (N, 3) array of points.
        
    Returns:
        T: (4, 4) SE(3) transformation matrix.
    """
    # Step 1: Compute centroid as median
    centroid = np.median(point_cloud, axis=0)
    
    # Step 2: Translate point cloud to centroid
    translated_point_cloud = point_cloud - centroid
    
    # Step 3: Compute covariance matrix
    covariance_matrix = np.cov(translated_point_cloud, rowvar=False)
    
    # Step 4: Compute eigenvectors and eigenvalues
    eigenvalues, eigenvectors = np.linalg.eigh(covariance_matrix)
    
    # Step 5: Sort eigenvectors by eigenvalues in descending order
    # The z-axis is the principal axis with the smallest eigenvalue
    sort_indices = eigenvalues.argsort()[::-1]
    eigenvectors = eigenvectors[:, sort_indices]
    
    # Step 6: Ensure right-handed coordinate system
    if np.linalg.det(eigenvectors) < 0:
        eigenvectors[:, 0] *= -1
        
    # Step 7: Create rotation matrix and SE(3) transform
    rotation_matrix = eigenvectors.T
    transform = np.eye(4)
    transform[:3, :3] = rotation_matrix
    transform[:3, 3] = -rotation_matrix @ centroid
    
    return transform
```
[S1, unofficial]

**Parameters**
- No tunable parameters. The method is deterministic given the input point cloud.

**Edge Cases and Pitfalls**
- **Degenerate Point Clouds**: If the point cloud is collinear or coplanar, the covariance matrix will be singular (eigenvalues will be zero). `np.linalg.eigh` will still return eigenvectors, but the orientation along the zero-variance axes is arbitrary. This may lead to unstable rotations in downstream optimization if not handled [G].
- **Determinant Check**: The check `np.linalg.det(eigenvectors) < 0` is critical. Without it, the resulting transform may include a reflection, which can break differentiable rendering pipelines that assume proper rotations [S1, unofficial].
- **Median vs Mean**: Using the median instead of the mean makes the centering robust to outliers. If outliers are present, the mean-based centering would shift the cloud incorrectly [S1, unofficial].

## Subtopics

### [Object Coarse Geometry Prediction](17357-object-coarse-geometry-prediction.md)
Object Coarse Geometry Prediction initializes the 6-DoF pose (rotation and translation) and scale of individual objects within a scene by fitting a parametric mesh to an approximate point cloud derived from image segmentation. It serves as the initialization step for differentiable rendering optimization, ensuring that subsequent gradient-based refinement starts from a geometrically plausible state rather than a random or poorly scaled configuration. It differs from sibling methods by explicitly decoupling scale initialization (via median-based radius matching) from rotational initialization (via discrete search over vertical-axis rotations), and by incorporating heuristic constraints for floor contact and floating objects to stabilize the optimization landscape.

## Operating characteristics
- **Time Complexity**: $O(N)$ for median and covariance computation (where $N$ is the number of points), and $O(1)$ for the $3 \times 3$ eigen decomposition.
- **Memory Complexity**: $O(N)$ to store the point cloud and intermediate centered points. The covariance matrix and eigenvectors are constant size ($3 \times 3$).
- **Strengths**: Robust to outliers due to median centering; provides a canonical orientation for the point cloud; simple and fast to compute.
- **Limitations**: Assumes the point cloud has a clear principal axis structure (e.g., elongated or planar objects). For spherical or isotropic objects, the principal axes are undefined or unstable.
- **When to Use**: When normalizing point clouds for downstream geometric processing, such as pose estimation or coarse geometry prediction.
- **When Not to Use**: When the point cloud lacks a dominant orientation or when outliers are not the primary concern (mean might be preferred for symmetric distributions).

## Background
Principal Axes Alignment is a standard technique in point cloud processing and computer vision. It is used to normalize the orientation and position of 3D data. In the context of the provided source, it is part of the preprocessing pipeline for 3D Gaussian Splatting optimization, specifically to initialize the pose and scale of objects in a scene [S1, unofficial]. The method relies on the statistical properties of the point cloud (covariance) to determine its dominant orientation.

## Sources
[S1] meshroomHub/mrGSplat, https://github.com/meshroomHub/mrGSplat, fetched 2026-10-27 (unofficial)
