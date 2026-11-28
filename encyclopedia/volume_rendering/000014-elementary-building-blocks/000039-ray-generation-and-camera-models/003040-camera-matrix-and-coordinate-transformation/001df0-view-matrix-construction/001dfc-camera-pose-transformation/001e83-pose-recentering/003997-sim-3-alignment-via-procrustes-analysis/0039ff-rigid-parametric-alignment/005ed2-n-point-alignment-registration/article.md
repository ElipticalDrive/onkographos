# N-Point Alignment Registration

Also called: Automatic n-point alignment, Least-square approximation registration

## Overview
N-Point Alignment Registration is an automatic tool that aligns source and destination objects by finding a transformation that minimizes the error between corresponding landmarks [S1]. It solves the problem of registering volumetric objects when manual transformation is difficult, using a least-square approximation based on user-specified points [S1]. It differs from its parent [Rigid Parametric Alignment](8609-rigid-parametric-alignment.md) by relying on explicit landmark correspondences rather than closed-form SVD minimization of reprojection error, and from [Iterative Closest Point (ICP)](10563-iterative-closest-point-icp.md) by requiring manual point selection rather than automatic nearest-neighbor matching [S1].

## How it works
**Inputs and Outputs:**
*   **Inputs:** A set of $N$ corresponding landmark points in the source object ($P_{source} = \{p_{s,i}\}_{i=1}^N$) and the destination object ($P_{dest} = \{p_{d,i}\}_{i=1}^N$), where $N \ge 4$ [S1].
*   **Outputs:** A transformation matrix $T$ that aligns the source to the destination [S1].

**Data Structures:**
*   Landmarks are stored as 3D coordinates. The source and destination sets must have equal cardinality $N$ [S1].

**Steps:**
1.  Identify at least four corresponding landmarks in both the source and destination objects [S1].
2.  Compute the transformation using the least-square approximation method based on the specified landmarks [S1].

**Parameters:**
*   **Minimum landmarks:** 4 corresponding points are required [S1].
*   **Landmark count:** More landmarks specified result in a better fit [S1].

**Pitfalls:**
*   The method requires manual identification of landmarks, which can be tedious for complex geometries [S1].
*   Precision depends on the accuracy of landmark placement [S1].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def n_point_alignment(source_landmarks, dest_landmarks):
    """
    Computes the transformation aligning source_landmarks to dest_landmarks
    using least-square approximation.
    
    Args:
        source_landmarks: List of N tuples (x, y, z) representing source points.
        dest_landmarks: List of N tuples (x, y, z) representing destination points.
        
    Returns:
        Transformation matrix T (4x4 homogeneous matrix) aligning source to dest.
    """
    # Source requires at least 4 corresponding landmarks [S1]
    if len(source_landmarks) < 4 or len(dest_landmarks) < 4:
        raise ValueError("At least 4 corresponding landmarks are required.")
    
    if len(source_landmarks) != len(dest_landmarks):
        raise ValueError("Source and destination landmark counts must match.")
    
    # Convert to numpy arrays for matrix operations
    P_src = np.array(source_landmarks)
    P_dst = np.array(dest_landmarks)
    
    # Compute centroids
    centroid_src = np.mean(P_src, axis=0)
    centroid_dst = np.mean(P_dst, axis=0)
    
    # Center the points
    P_src_centered = P_src - centroid_src
    P_dst_centered = P_dst - centroid_dst
    
    # Compute the covariance matrix H
    H = P_src_centered.T @ P_dst_centered
    
    # Singular Value Decomposition to find rotation
    U, S, Vt = np.linalg.svd(H)
    
    # Calculate rotation matrix R
    # Check for reflection
    d = np.linalg.det(Vt.T @ U.T)
    if d < 0:
        Vt[-1, :] *= -1
        R = Vt.T @ U.T
    else:
        R = Vt.T @ U.T
        
    # Calculate translation vector t
    t = centroid_dst - R @ centroid_src
    
    # Construct homogeneous transformation matrix
    T = np.eye(4)
    T[:3, :3] = R
    T[:3, 3] = t
    
    return T
```
[G] The pseudocode above implements the standard Kabsch algorithm, which is the standard least-square solution for rigid point set registration. While [S1] mentions "least-square approximation," it does not specify the exact linear algebra steps. The Kabsch algorithm is the canonical implementation for minimizing the root-mean-square deviation (RMSD) between two sets of points, which aligns with the description of minimizing error between corresponding landmarks.

## Subtopics

### Image-Based Rigid Alignment via Gradient Descent
Aligns EM image tiles acquired at different scales and spatial locations into a single coordinate system using a fixed-size reference grid [S2, unofficial]. It differs from standard n-point alignment by operating on pixel intensities rather than explicit landmarks, using gradient descent to minimize image difference energy [S2, unofficial]. Choose this when registering multi-scale microscopy data where explicit landmarks are difficult to identify [S2, unofficial].

*   **Definition:** Aligns EM image tiles acquired at different scales and spatial locations into a single coordinate system using a fixed-size reference grid [S2, unofficial].
*   **Steps:**
    1.  Select a lower-level image as the background (reference) image $I_L$ and a higher-level image as the target image $I_H$ [S2, unofficial].
    2.  Sub/super-sample each image based on its magnification level relative to the reference grid resolution [S2, unofficial].
    3.  Iteratively update transformation parameters (translation vector, rotation angle, zoom factor) using gradient descent along the negative gradient direction [S2, unofficial].
*   **Formula:** Minimize the image difference energy $E = \sum_{i=1}^{n} ||I_L(x_i) - (T \cdot I_H)(x_i)||^2$, where $T$ is the rigid transformation, $x_i$ is the i-th pixel index on the reference grid, and $I(x_i)$ is the pixel value of image $I$ at $x_i$ [S2, unofficial].
*   **Parameters:** Reference grid size: 256 × 256 pixels [S2, unofficial].
*   **Pitfalls:** Registration computation time is independent of the image tile size because computation is done on the fixed reference grid [S2, unofficial].
*   **Performance:** Registration running time: ~3 ms per single run on an NVIDIA Fermi GPU (GTX 580) [S2, unofficial].

## Operating characteristics
*   **Time Complexity:** Not given in the sources.
*   **Memory Complexity:** Not given in the sources.
*   **Measured Performance:** Not given in the sources for the general n-point method. The variant using gradient descent runs in ~3 ms per single run on an NVIDIA Fermi GPU (GTX 580) [S2, unofficial].
*   **Scaling:** The more landmarks specified, the better the fit, implying computational cost increases with $N$ [S1].
*   **Strengths:** Automatic registration for difficult cases; improves fit with more landmarks [S1].
*   **Limitations:** Requires manual identification of at least 4 corresponding landmarks [S1].
*   **When to use:** When manual transformation is difficult and corresponding landmarks can be identified [S1].
*   **When not to use:** When landmarks are ambiguous or cannot be reliably identified [S1].

## Background
Originated in the VIVIAN system for multimodal volume-based tumor neurosurgery planning [S1]. It is used to align volumetric objects, such as tumors and vessels, for intersection analysis [S1].

## Sources
[S1] Multimodal volume-based tumor neurosurgery planning in the virtual workbench, https://doi.org/10.1007/bfb0056289, fetched 2026-10-26
[S2] Petascale Volume Visualization of Microscopy Data Streams, https://vcg.seas.harvard.edu/publications/20130101-exploring-the-connectome-petascale-volume-visualization-of-microscopy-data-streams/paper, fetched 2026-10-27 (unofficial)
