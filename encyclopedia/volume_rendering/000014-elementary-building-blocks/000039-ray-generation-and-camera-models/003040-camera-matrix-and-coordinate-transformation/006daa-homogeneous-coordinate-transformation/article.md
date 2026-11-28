# Homogeneous Coordinate Transformation

Also called: Homogeneous Transformation, Homogenization

## Overview
Homogeneous Coordinate Transformation maps points, vectors, and rays between coordinate spaces by embedding 3D entities into 4D homogeneous space, applying a $4 \times 4$ transformation matrix, and extracting the result [S1, unofficial]. It solves the problem of applying affine transformations uniformly to positions and directions while preventing translation from affecting vector orientations [G], and differs from [NDC Ray Transformation](905-ndc-ray-transformation.md) by handling general affine basis changes rather than canonical cube mapping [G].

## How it works
**Inputs:**
- A $4 \times 4$ transformation matrix $M$ (or its inverse $M^{-1}$) [S1, unofficial].
- A set of 3D points $\mathbf{p} = (x, y, z)$, ray origins, or directions $\mathbf{d} = (d_x, d_y, d_z)$ [S1, unofficial].

**Outputs:**
- Transformed 3D points $\mathbf{p}'$, ray origins, or directions $\mathbf{d}'$ [S1, unofficial].

**Data Structures:**
- A homogeneous vector $\mathbf{v}_h \in \mathbb{R}^4$ [G].
- For points and ray origins: $\mathbf{v}_h = [x, y, z, 1]^T$ [S1, unofficial].
- For directions: $\mathbf{v}_h = [d_x, d_y, d_z, 0]^T$ [S1, unofficial].

**Steps:**
1. **Homogenization**: Convert the 3D input into a 4D homogeneous vector by appending $w=1$ for points and ray origins, or $w=0$ for directions [S1, unofficial].
2. **Matrix Multiplication**: Multiply the homogeneous vector by the transformation matrix $M$ [S1, unofficial].
3. **Extraction**: Extract the first three components $(x', y', z')$ of the resulting vector $\mathbf{v}'_h$ [S1, unofficial].

**Formulas:**
Let $M$ be:
$$
M = \begin{bmatrix}
m_{00} & m_{01} & m_{02} & m_{03} \\
m_{10} & m_{11} & m_{12} & m_{13} \\
m_{20} & m_{21} & m_{22} & m_{23} \\
m_{30} & m_{31} & m_{32} & m_{33}
\end{bmatrix}
$$
For a point or ray origin $\mathbf{p} = (x, y, z)$:
$$
\begin{bmatrix} x' \\ y' \\ z' \\ 1 \end{bmatrix} = M \begin{bmatrix} x \\ y \\ z \\ 1 \end{bmatrix}
$$
$$
\begin{aligned}
x' &= m_{00}x + m_{01}y + m_{02}z + m_{03} \\
y' &= m_{10}x + m_{11}y + m_{12}z + m_{13} \\
z' &= m_{20}x + m_{21}y + m_{22}z + m_{23}
\end{aligned}
$$
For a direction $\mathbf{d} = (d_x, d_y, d_z)$:
$$
\begin{bmatrix} d'_x \\ d'_y \\ d'_z \\ 0 \end{bmatrix} = M \begin{bmatrix} d_x \\ d_y \\ d_z \\ 0 \end{bmatrix}
$$
$$
\begin{aligned}
d'_x &= m_{00}d_x + m_{01}d_y + m_{02}d_z \\
d'_y &= m_{10}d_x + m_{11}d_y + m_{12}d_z \\
d'_z &= m_{20}d_x + m_{21}d_y + m_{22}d_z
\end{aligned}
$$

**Parameters:**
- $M$: The transformation matrix, typically derived from [View Matrix Construction](1644-view-matrix-construction.md) [G].

**Edge Cases and Pitfalls:**
- **Non-affine transformations**: If $M$ contains perspective components (non-zero $m_{30}, m_{31}, m_{32}$), the resulting $w'$ component will not be $1$ or $0$, requiring perspective division ($x'/w', y'/w', z'/w'$) [G].
- **Inverse Matrices**: To transform from object-space to world-space, use $M^{-1}$ instead of $M$ [S1, unofficial].
- **Precision**: Floating-point errors in matrix inversion can accumulate, requiring numerically stable inversion methods for high precision [G].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import numpy as np

def transform_homogeneous(entities, matrix, is_direction=False):
    """
    Transform 3D points, ray origins, or directions using a 4x4 homogeneous matrix.
    
    Args:
        entities: Array of shape (N, 3) containing x, y, z.
        matrix: 4x4 numpy array representing the transformation.
        is_direction: Boolean. True if inputs are directions (w=0), False if points/origins (w=1).
        
    Returns:
        Transformed array of shape (N, 3).
    """
    # Step 1: Homogenization
    # Create w component: 1.0 for points/origins, 0.0 for directions
    w = np.zeros((entities.shape[0], 1)) if is_direction else np.ones((entities.shape[0], 1))
    # Concatenate to form homogeneous coordinates [x, y, z, w]
    homogeneous_coords = np.hstack([entities, w])
    
    # Step 2: Matrix Multiplication
    # Multiply by matrix. Using row-major storage: v' = v @ M.T
    transformed_homogeneous = homogeneous_coords @ matrix.T
    
    # Step 3: Extraction
    # Extract first three components. For affine transforms, w is preserved.
    transformed_3d = transformed_homogeneous[:, :3]
    
    return transformed_3d
```

## Subtopics

### [Coordinate Transformations](22961-coordinate-transformations.md)
Deterministic functions mapping input spatial coordinates to a transformed domain for factor fields. Sets itself apart by handling internal neural representation mappings (e.g., positional encoding) rather than camera-to-world projection. Choose when using Factor Fields frameworks.

### [NDC Ray Transformation](905-ndc-ray-transformation.md)
Maps rays to Normalized Device Coordinates. Sets itself apart by targeting the canonical cube $[-1, 1]^3$ for uniform sampling. Choose for forward-facing scene rendering.

### [View Matrix Construction](1644-view-matrix-construction.md)
Builds the world-to-view matrix. Sets itself apart by deriving the matrix from camera basis vectors. Choose when constructing the transformation matrix itself.

### [Point to Camera Transformation](1660-point-to-camera-transformation.md)
Transforms points to camera space. Sets itself apart by focusing specifically on point positions for visibility checks. Choose for occlusion testing.

### [Arbitrary Source-Detector Geometry Projector](2975-arbitrary-source-detector-geometry-projector.md)
Handles non-standard projector geometries. Sets itself apart by supporting arbitrary source-detector configurations. Choose for custom optical setups.

### [Gaussian Transformation to World Space](6432-gaussian-transformation-to-world-space.md)
Transforms Gaussian splats to world space. Sets itself apart by handling covariance matrices. Choose for 3D Gaussian Splatting.

### [Affine Transformation for Glass Facet Mapping](14529-affine-transformation-for-glass-facet-mapping.md)
Specific affine transforms for glass facets. Sets itself apart by optimizing for refractive surfaces. Choose for glass rendering.

## Operating characteristics
- **Time Complexity**: $O(N)$ for $N$ entities, with a constant factor of 16 multiplications and 12 additions per entity for affine transforms [G].
- **Memory Complexity**: $O(N)$ to store the homogeneous coordinates temporarily [G].
- **Performance**: Highly optimized on GPUs due to parallel matrix-vector multiplication [G].
- **Strengths**: Unified handling of points and vectors; mathematically robust for affine transforms [G].
- **Limitations**: Requires explicit handling of the $w$ component; not suitable for non-affine transforms without perspective division [G].
- **When to use**: Standard coordinate space conversions in ray tracing [G].
- **When not to use**: When perspective division is required, use [NDC Ray Transformation](905-ndc-ray-transformation.md) or explicit perspective handling [G].

## Background
Originated in computer graphics to unify affine transformations across different coordinate systems [G]. The method is implemented in libraries like `gradoptics` for ray generation and coordinate transformation [S1, unofficial].

## Sources
[S1] magis-slac/gradoptics, https://github.com/magis-slac/gradoptics, fetched 2026-10-27 (unofficial)
