# Average Pose Calculation
Also called: Average Pose Computation, mean pose via SVD
## Overview
Computes a representative camera pose from a set of input camera poses by averaging their translations and rotations. It differs from simple vector averaging by using Singular Value Decomposition (SVD) to ensure the resulting rotation matrix remains orthogonal and valid, addressing the non-linear nature of rotation space [S2, unofficial]. This method is primarily used to initialize a central view or create a canonical coordinate system for a scene [S1, S2, unofficial].

## How it works
**Inputs:**
- `poses`: A list or array of $N$ camera poses. Each pose is a $3 \times 4$ or $4 \times 4$ transformation matrix (camera-to-world). The first $3 \times 3$ columns represent rotation, and the last column represents translation [S2, unofficial].

**Outputs:**
- `avg_pose`: A single $3 \times 4$ or $4 \times 4$ matrix representing the average pose [S2, unofficial].

**Data Structures:**
- `poses`: Shape $(N, 3, 4)$ or $(N, 4, 4)$.
- `center`: Shape $(3,)$, the averaged translation vector.
- `mean_rot`: Shape $(3, 3)$, the element-wise mean of the rotation matrices.
- `avg_rot`: Shape $(3, 3)$, the orthogonalized average rotation matrix.

**Steps:**
1. **Extract Translation:** Calculate the average translation vector `center` by taking the mean of the translation columns (index 3) of all poses along the batch axis [S2, unofficial].
   $$ \text{center} = \frac{1}{N} \sum_{i=1}^{N} \text{poses}_i[:, 3] $$
2. **Compute Mean Rotation:** Calculate the element-wise mean of the rotation sub-matrices (indices 0-2) of all poses [S2, unofficial].
   $$ \text{mean\_rot} = \frac{1}{N} \sum_{i=1}^{N} \text{poses}_i[:, :3, :3] $$
3. **Orthogonalize Rotation:** Perform Singular Value Decomposition (SVD) on `mean_rot` to obtain matrices $U$ and $V^T$ [S2, unofficial].
   $$ U, \Sigma, V^T = \text{SVD}(\text{mean\_rot}) $$
   Compute the average rotation matrix `avg_rot` as:
   $$ \text{avg\_rot} = U V^T $$
   *Note:* If $\det(U V^T) = -1$, a correction is typically required to ensure a proper rotation (no reflection), though the sources do not explicitly detail this reflection handling [S2, unofficial].
4. **Construct Average Pose:** Concatenate `avg_rot` and `center` (reshaped to a column vector) to form the final $3 \times 4$ pose matrix [S2, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import numpy as np

def average_pose_calculation(poses):
    """
    Compute the average camera pose from a set of poses using SVD for rotation averaging.
    
    Args:
        poses: numpy array of shape (N, 3, 4) or (N, 4, 4) containing camera-to-world matrices.
        
    Returns:
        avg_pose: numpy array of shape (3, 4) or (4, 4) representing the average pose.
    """
    # Ensure poses are 3x4 for consistency
    if poses.shape[-2] == 4:
        poses = poses[:, :3, :]
        
    N = poses.shape[0]
    
    # Step 1: Calculate average translation (center)
    # Mean of the translation columns (index 3)
    center = np.mean(poses[:, :, 3], axis=0)
    
    # Step 2: Calculate mean rotation
    # Mean of the rotation sub-matrices (first 3 columns)
    mean_rot = np.mean(poses[:, :, :3], axis=0)
    
    # Step 3: Orthogonalize rotation using SVD
    # Perform SVD on the mean rotation matrix
    U, _, Vt = np.linalg.svd(mean_rot)
    
    # Compute the orthogonal average rotation
    avg_rot = U @ Vt
    
    # Step 4: Construct the average pose matrix
    # Concatenate rotation and translation
    avg_pose = np.hstack((avg_rot, center.reshape(-1, 1)))
    
    return avg_pose
```

**Parameters:**
- `poses`: Input poses, either $[N, 3, 4]$ or $[N, 4, 4]$ [S2, unofficial].

**Edge Cases and Pitfalls:**
- **Reflection:** If the determinant of $U V^T$ is $-1$, the result is a reflection, not a rotation. Standard implementations often correct this by flipping the sign of the last column of $U$ or $V^T$, but this specific correction is not detailed in the provided sources [S2, unofficial].
- **Degenerate Input:** If all poses are identical or collinear in rotation space, the mean rotation may be singular, leading to unstable SVD results [G].
- **Precision:** Floating-point errors in the mean rotation matrix can lead to slight non-orthogonality before SVD, which SVD corrects [S2, unofficial].

## Operating characteristics
**Time Complexity:** $O(N \cdot d^2)$ where $N$ is the number of poses and $d=3$ is the dimensionality of the rotation matrix. The dominant operation is SVD, which is constant time for fixed $3 \times 3$ matrices.
**Memory Complexity:** $O(N \cdot d^2)$ to store the input poses and intermediate mean rotation matrix.
**Strengths:** Ensures the resulting rotation is valid (orthogonal) unlike simple averaging [S2, unofficial]. Simple to implement using standard linear algebra libraries [S2, unofficial].
**Limitations:** Does not account for the geodesic distance on the rotation manifold (SO(3)), which can be less accurate for large angular differences between poses compared to iterative methods [G].
**When to use:** When a quick, central reference pose is needed for initialization or alignment [S1, S2, unofficial].
**When not to use:** When high precision in rotation averaging is required for widely varying camera orientations [G].

## Background
The method originates from implementations in NeRF (Neural Radiance Fields) pipelines, specifically for computing a canonical camera pose from a dataset of images [S1, S2, unofficial]. It is commonly referred to as `poses_avg` in codebases like `bmild/nerf` and `nerf-from-scratch` [S1, S2, unofficial]. The use of SVD for rotation averaging is a standard technique in computer vision for handling rotation matrices [S2, unofficial].

## Sources
[S1] bmild/nerf, https://github.com/bmild/nerf, fetched 2026-10-26 (unofficial)
[S2] damn8daniel/nerf-from-scratch, https://github.com/damn8daniel/nerf-from-scratch, fetched 2026-10-26 (unofficial)
