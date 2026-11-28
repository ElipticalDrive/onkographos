# Model-View Transformation

Also called: Model Matrix Construction, Object Transformation, View Vector Calculation, updateViewVector

## Overview
The Model-View Transformation computes the normalized view direction vector from the camera to the mesh by multiplying the camera's world inverse matrix with the mesh's world matrix, then extracting the negative Z-axis components [S2, unofficial]. It differs from the sibling [View Matrix Construction](1644-view-matrix-construction.md) by focusing on extracting the view vector from the composite transformation rather than constructing the full camera-to-world matrix [S2, unofficial].

## How it works
**Inputs:**
1. `camera.matrixWorldInverse`: A 4x4 matrix representing the inverse of the camera's world transformation [S2, unofficial].
2. `mesh.matrixWorld`: A 4x4 matrix representing the world transformation of the target object [S2, unofficial].

**Outputs:**
1. `modelViewMatrix`: A 4x4 matrix computed from the camera's inverse world matrix and the mesh's world matrix [S2, unofficial].
2. `viewVector`: A normalized 3D vector pointing from the camera towards the object, derived from the negative Z-axis components of the model-view matrix [S2, unofficial].

**Data Structures:**
* **Matrix**: 4x4 array of 16 floating-point values stored in a flat `elements` array indexed 0–15 [S2, unofficial].
* **Vector**: 3D vector $(x, y, z)$ [S2, unofficial].

**Steps:**
1. Initialize `modelViewMatrix` to the identity matrix [S1, unofficial].
2. Compute the product of the camera's inverse world matrix and the mesh's world matrix: $M_{mv} = M_{cam}^{-1} \times M_{mesh}$ [S2, unofficial].
3. Extract the components at indices 2, 6, and 10 from the flat matrix array [S2, unofficial].
4. Negate these components to obtain the direction from the camera to the object: $v_x = -M_{mv}[2]$, $v_y = -M_{mv}[6]$, $v_z = -M_{mv}[10]$ [S2, unofficial].
5. Normalize the resulting vector $(v_x, v_y, v_z)$ to unit length [S2, unofficial].

**Formulas:**
The transformation matrices for translation, rotation, and scaling are defined as follows [S1, unofficial]:

Translation by $(x, y, z)$:
$$ T = \begin{pmatrix} 1 & 0 & 0 & -x \\ 0 & 1 & 0 & -y \\ 0 & 0 & 1 & -z \\ 0 & 0 & 0 & 1 \end{pmatrix} $$

Rotation of $\theta$ degrees about a unit vector axis $(x, y, z)$ (specifically about the X-axis in the source example):
$$ R = \begin{pmatrix} 1 & 0 & 0 & 0 \\ 0 & \cos(\theta) & -\sin(\theta) & 0 \\ 0 & \sin(\theta) & \cos(\theta) & 0 \\ 0 & 0 & 0 & 1 \end{pmatrix} $$

Scaling by $(a, b, c)$:
$$ S = \begin{pmatrix} a & 0 & 0 & 0 \\ 0 & b & 0 & 0 \\ 0 & 0 & c & 0 \\ 0 & 0 & 0 & 1 \end{pmatrix} $$

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def compute_model_view_transformation(camera_matrix_world_inverse, mesh_matrix_world):
    # Step 2: Multiply camera inverse world matrix by mesh world matrix
    model_view_matrix = multiply_matrices(camera_matrix_world_inverse, mesh_matrix_world)
    # Step 3 & 4: Extract components at indices 2, 6, 10 and negate
    vx = -model_view_matrix[2]
    vy = -model_view_matrix[6]
    vz = -model_view_matrix[10]
    # Step 5: Normalize
    length = (vx * vx + vy * vy + vz * vz) ** 0.5
    if length == 0.0:
        return (0.0, 0.0, 0.0)
    return (vx / length, vy / length, vz / length)

def multiply_matrices(A, B):
    # A and B are 4x4 matrices stored as flat lists of 16 floats
    C = [0.0] * 16
    for i in range(4):
        for j in range(4):
            s = 0.0
            for k in range(4):
                s += A[i * 4 + k] * B[k * 4 + j]
            C[i * 4 + j] = s
    return C
```

**Edge Cases and Pitfalls:**
* **Degenerate Camera/Mesh Pose:** If the camera and mesh are in identical positions or the transformation results in a zero vector, normalization fails [G].
* **Matrix Layout:** The indices 2, 6, 10 depend on the specific memory layout (row-major vs column-major) and the definition of the matrix multiplication order [G].
* **Precision:** Floating-point errors in matrix multiplication can lead to slight non-orthogonality in the basis vectors [G].
* **Implementation Note:** Implementers must verify the index mapping against their matrix library's storage order [G].

## Operating characteristics
* **Time Complexity:** $O(1)$ for matrix multiplication of fixed 4x4 matrices and vector normalization [G].
* **Memory Complexity:** $O(1)$, storing a few 4x4 matrices and a 3D vector [G].
* **Strengths:** Directly provides the view vector needed for lighting calculations without requiring additional vector math from the camera position and object position [G].
* **Limitations:** Tightly coupled to the specific matrix layout and indexing scheme of the source implementation (indices 2, 6, 10) [G].
* **When to use:** When the view vector relative to a specific mesh is required for per-vertex or per-pixel lighting in a volumetric or surface rendering context [G].
* **When not to use:** When only the camera's global orientation is needed, or when working with a different matrix layout that maps the Z-axis to different indices [G].

## Background
The modelview matrix is initialized to the identity matrix and multiplied by translation, rotation, or scaling matrices as the scene moves [S1, unofficial]. The specific implementation of extracting the view vector from the model-view matrix is documented in the context of volumetric fire visualization using Three.js [S2, unofficial]. The underlying matrix definitions for translation, rotation, and scaling are standard OpenGL conventions [S1, unofficial].

## Sources
[S1] Smokeview (Version 6) A Tool for Visualizing Fire Dynamics Simulation Data Volume II : Technical Reference Guide, https://doi.org/10.6028/nist.sp.1017-2, fetched 2026-10-26 (unofficial)
[S2] yomotsu/VolumetricFire, https://github.com/yomotsu/VolumetricFire, fetched 2026-10-27 (unofficial)
