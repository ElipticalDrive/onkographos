# Camera Matrix Decomposition
Also called: Decompose Projection Matrix, Intrinsic/Extrinsic Decomposition, PnP Decomposition
## Overview
This method decomposes a $3 \times 4$ camera projection matrix $P$ into its constituent intrinsic parameters (focal length, principal point, skew), extrinsic rotation $R$, and translation $t$ [S1, unofficial]. It solves the problem of recovering the physical camera configuration from a projective transformation matrix, enabling the initialization of metric reconstruction pipelines [S1, unofficial]. It differs from non-linear bundle adjustment by providing a direct, linear initial estimate via Singular Value Decomposition (SVD) constraints, rather than iterative optimization [S1, unofficial].
## How it works
**Inputs:** A $3 \times 4$ projection matrix $P$ (float32).
**Outputs:** Intrinsic matrix $K$ ($3 \times 3$), Rotation matrix $R$ ($3 \times 3$), Translation vector $t$ ($3 \times 1$).

**Data Structures:**
- $P$: $3 \times 4$ matrix.
- $K$: $3 \times 3$ upper triangular matrix.
- $R$: $3 \times 3$ orthogonal matrix ($R^T R = I$).
- $t$: $3 \times 1$ vector.

**Steps:**
1. **Decomposition:** Compute the decomposition of $P$ into $K$, $R$, and a homogeneous translation vector $T_h$. The standard decomposition satisfies $P = K [R | t]$. Note that $P$ is defined up to scale.
2. **Intrinsic Normalization:** The extracted intrinsic matrix $K_{raw}$ must be normalized. Divide $K_{raw}$ by its element $K_{raw}[2,2]$ to ensure the bottom-right element is 1.0.
   $$ K = \frac{K_{raw}}{K_{raw}[2,2]} $$
3. **Rotation Extraction:** The rotation matrix $R$ is extracted directly. Ensure it is a proper rotation (determinant +1) by checking $\det(R)$. If $\det(R) = -1$, negate $R$ and $t$ to correct the coordinate system [G].
4. **Translation Extraction:** The translation vector $t$ is derived from the homogeneous translation $T_h$ (often the last column of the decomposed matrix or a specific output index). Normalize $t$ by dividing by its homogeneous component $T_h[3]$ (if represented in homogeneous coordinates) or simply taking the first three components if the decomposition routine returns Euclidean translation directly.
   $$ t = \frac{T_h[0:3]}{T_h[3]} $$
5. **Pose Matrix Construction:** Construct the $4 \times 4$ pose matrix $M_{pose}$ for use in rendering pipelines:
   $$ M_{pose} = \begin{bmatrix} R^T & -R^T t \\ 0 & 1 \end{bmatrix} $$
   Note: The rotation is transposed because $R$ typically transforms world-to-camera, while pose matrices often define camera-to-world or vice-versa depending on the convention. The translation is adjusted to represent the camera center.

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import numpy as np

def decompose_projection_matrix(P):
    """
    Decomposes a 3x4 projection matrix P into K, R, t.
    P: numpy array, shape (3, 4), dtype float32.
    Returns: K (3x3), R (3x3), t (3x1)
    """
    # Step 1: Decompose using standard algorithm (e.g., OpenCV's decomposeProjectionMatrix)
    # The function returns: (K, R, T, Q, D, E, F)
    # K is at index 0, R at index 1, T (homogeneous translation) at index 2
    K_raw, R, T_h, *_ = decompose_projection_matrix_opencv(P)
    
    # Step 2: Normalize Intrinsics
    # Ensure K[2,2] is 1.0
    K = K_raw / K_raw[2, 2]
    
    # Step 3: Handle Rotation Sign Ambiguity
    # Check determinant to ensure proper rotation
    if np.linalg.det(R) < 0:
        R = -R
        T_h = -T_h
        
    # Step 4: Extract Translation
    # T_h is typically a 4x1 vector or the last column of the 3x4 matrix in some contexts.
    # If T_h is the 3-vector from the decomposition routine (as per source note):
    # Source note says: "translation divided by translation[3]"
    # This implies T_h might be 4D or we need to handle the scale.
    # Standard OpenCV decomposeProjectionMatrix returns T as a 3-vector in some versions, 
    # but the source note explicitly mentions dividing by translation[3].
    # Assuming T_h is 4D homogeneous vector [tx, ty, tz, w]
    if T_h.shape[0] == 4:
        t = T_h[:3] / T_h[3]
    else:
        # Fallback if T_h is 3D (scale ambiguity remains)
        t = T_h
        
    return K, R, t

def decompose_projection_matrix_opencv(P):
    """
    Placeholder for OpenCV's cv2.decomposeProjectionMatrix behavior.
    Returns K (3x3), R (3x3), T (4x1 homogeneous), and other matrices.
    """
    # Implementation detail: This function performs QR decomposition or SVD based extraction.
    # P = K [R | t]
    # We extract K by QR decomposition of the first 3x3 of P? 
    # Actually, standard method:
    # 1. Compute QR decomposition of P[:, :3]
    # 2. Extract K and R.
    # 3. Solve for t.
    # For the purpose of this article, we assume a black-box function that returns the standard outputs.
    pass
```

**Parameters:**
- Precision: float32 is required for stability in the decomposition step [S1, unofficial].

**Edge Cases:**
- **Singular K:** If $K$ is singular (e.g., zero focal length), decomposition fails.
- **Scale Ambiguity:** The projection matrix is scale-invariant. The decomposition returns $t$ up to a scale factor. The normalization $K[2,2]=1$ fixes the scale of $K$, but $t$'s magnitude depends on the scene scale.
- **Degenerate Input:** If $P$ is not a valid projection matrix (rank < 3), decomposition may produce invalid $R$ or $K$.

## Subtopics

### [Self-Calibration via Absolute Dual Quadric](4732-self-calibration-via-absolute-dual-quadric.md)
Recovers metric reconstruction from projective reconstruction by imposing constraints on the absolute dual quadric $\Omega^*$. Sets itself apart by formulating calibration as a linear system solvable via SVD, providing an initial estimate for refinement.

## Operating characteristics
- **Time Complexity:** $O(1)$ for fixed-size $3 \times 4$ matrix decomposition, dominated by SVD or QR decomposition steps.
- **Memory Complexity:** $O(1)$, storing only $3 \times 4$ and $3 \times 3$ matrices.
- **Strengths:** Fast, deterministic, provides a good initial guess for non-linear optimization.
- **Limitations:** Sensitive to noise in $P$; assumes a valid pinhole camera model; scale ambiguity in translation.
- **When to Use:** Initializing camera parameters for volume rendering or 3D reconstruction from known projection matrices.
- **When Not to Use:** When the camera model is complex (e.g., fisheye) or when high precision is required without subsequent refinement.

## Background
Originating from computer vision literature, this method is a standard step in camera calibration and structure-from-motion pipelines. It is widely used in libraries like OpenCV [S1, unofficial]. In volume rendering, it is used to convert camera poses from tracking systems or reconstruction algorithms into renderable camera matrices.

## Sources
[S1] albertgassol1/vf-nerf, https://github.com/albertgassol1/vf-nerf, fetched 2026-10-26 (unofficial)
