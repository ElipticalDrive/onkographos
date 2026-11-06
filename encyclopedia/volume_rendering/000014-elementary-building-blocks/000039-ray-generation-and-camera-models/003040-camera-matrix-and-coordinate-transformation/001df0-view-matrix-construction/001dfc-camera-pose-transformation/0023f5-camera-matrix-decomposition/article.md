# Camera Matrix Decomposition
Also called: Decompose Projection Matrix, Intrinsic/Extrinsic Decomposition, PnP Decomposition
## Overview
Decomposes a 3x4 projection matrix into intrinsic matrix $K$, rotation matrix $R$, and translation vector $t$ [S1, unofficial]. It extracts the camera's internal parameters and external pose from a combined projection matrix, enabling the separation of optical properties from physical camera position and orientation [S1, unofficial].
## How it works
Inputs: A 3x4 projection matrix $P$ (float32). Outputs: Intrinsic matrix $K$ (3x3), Rotation matrix $R$ (3x3), Translation vector $t$ (3x1).

1. Load the projection matrix $P$ ensuring float32 type [S1, unofficial].
2. Decompose $P$ using a standard library function (e.g., OpenCV's `decomposeProjectionMatrix`) to obtain intermediate components: intrinsics $K'$, rotation $R'$, and translation $t'$ [S1, unofficial].
3. Normalize the intrinsic matrix by dividing by $K'[2,2]$ to ensure the bottom-right element is 1 [S1, unofficial].
4. Pad the normalized 3x3 intrinsic matrix to a 4x4 identity matrix for compatibility with homogeneous coordinate systems [S1, unofficial].
5. Construct the final pose matrix: transpose $R'$ to get $R$ (assuming column-major convention where $R$ rotates world to camera) [S1, unofficial].
6. Normalize the translation vector by dividing by $t'[3]$ to handle homogeneous scaling [S1, unofficial].

Pseudocode:
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import numpy as np

def decompose_projection_matrix(P: np.ndarray) -> tuple:
    """
    Decomposes a 3x4 projection matrix into K, R, t.
    :param P: 3x4 numpy array, float32
    :return: K (3x3), R (3x3), t (3x1)
    """
    # Step 1: Ensure float32
    P = P.astype(np.float32)
    
    # Step 2: Decompose using OpenCV
    # Note: cv2.decomposeProjectionMatrix returns (K, R, t, Q, ...). 
    # We assume the first three outputs are K, R, t as per source [S1, unofficial].
    import cv2
    K_prime, R_prime, t_prime, _ = cv2.decomposeProjectionMatrix(P)[:4]
    
    # Step 3: Normalize intrinsics
    K = K_prime / K_prime[2, 2]
    
    # Step 4: Pad to 4x4 identity (for internal use, though K is 3x3)
    # The source mentions padding to 4x4 identity, but typically K remains 3x3.
    # We return the 3x3 K as standard.
    
    # Step 5: Construct R (transpose R_prime)
    R = R_prime.T
    
    # Step 6: Normalize t
    t = t_prime / t_prime[3]
    
    return K, R, t
```
Edge cases: The source does not specify handling for singular matrices or degenerate camera poses [S1, unofficial]. Precision issues may arise if $P$ is not well-conditioned [G].
## Subtopics
## Operating characteristics
Time complexity: Depends on the underlying decomposition algorithm (e.g., QR decomposition in OpenCV), typically $O(1)$ for fixed 3x4 matrix [G]. Memory complexity: $O(1)$ for fixed-size matrices [G]. Strengths: Simple extraction of camera parameters [S1, unofficial]. Limitations: Relies on library implementation details; sign ambiguity in rotation/translation is not addressed in the source [G].
## Background
Origin: The method is implemented in the `vf-nerf` repository by albertgassol1 [S1, unofficial]. It uses OpenCV's `decomposeProjectionMatrix` function [S1, unofficial].
## Sources
[S1] albertgassol1/vf-nerf, https://github.com/albertgassol1/vf-nerf, fetched 2026-10-26 (unofficial)
