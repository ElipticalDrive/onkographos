# Custom Matrix Math Library

Also called: Native JavaScript Matrix Operations

## Overview
A custom implementation of 3D transformation matrices (perspective projection, look-at, translation, and scaling) written natively in JavaScript to eliminate external library dependencies [S1, unofficial]. It differs from sibling methods like [Camera Look-at Basis Construction](3990-camera-look-at-basis-construction.md) by providing complete, self-contained matrix generation functions rather than just basis vector computation, and from [General Perspective Transformation](1845-general-perspective-transformation.md) by using a specific column-major layout optimized for WebGL 2 shader inputs [S1, unofficial].

## How it works
**Inputs:**
- `fov`: Field of view angle in radians.
- `aspect`: Aspect ratio (width / height).
- `near`, `far`: Near and far clipping plane distances.
- `eye`: Camera position vector `[ex, ey, ez]`.
- `center`: Look-at target vector `[cx, cy, cz]`.
- `up`: Up vector `[ux, uy, uz]`.

**Outputs:**
- A `Float32Array` of length 16 representing a 4x4 matrix in column-major order.

**Data Structures:**
- Matrices are stored as `Float32Array([m0, m1, m2, m3, m4, m5, m6, m7, m8, m9, m10, m11, m12, m13, m14, m15])`.
- Indices 0-3: First column.
- Indices 4-7: Second column.
- Indices 8-11: Third column.
- Indices 12-15: Fourth column (translation/homogeneous).

**Steps:**

1.  **Perspective Projection Matrix Construction:**
    Calculate the focal length $f = 1.0 / \tan(\text{fov} / 2)$.
    Initialize output array to zeros.
    Set elements:
    $$ \text{out}[0] = f / \text{aspect} $$
    $$ \text{out}[5] = f $$
    $$ \text{out}[10] = (\text{far} + \text{near}) / (\text{near} - \text{far}) $$
    $$ \text{out}[11] = -1.0 $$
    $$ \text{out}[14] = (2.0 \times \text{far} \times \text{near}) / (\text{near} - \text{far}) $$
    Return `out` [S1, unofficial].

2.  **Look-At Matrix Construction:**
    Compute the forward vector $z = \text{eye} - \text{center}$.
    Normalize $z$: $z = z / \|z\|$.
    Compute the right vector $x = \text{up} \times z$.
    Normalize $x$: $x = x / \|x\|$.
    Compute the true up vector $y = z \times x$.
    Fill the rotation part of the matrix (columns 0, 1, 2):
    $$ \text{out}[0]=x_x, \text{out}[1]=y_x, \text{out}[2]=z_x, \text{out}[3]=0 $$
    $$ \text{out}[4]=x_y, \text{out}[5]=y_y, \text{out}[6]=z_y, \text{out}[7]=0 $$
    $$ \text{out}[8]=x_z, \text{out}[9]=y_z, \text{out}[10]=z_z, \text{out}[11]=0 $$
    Compute translation components to move the world relative to the camera:
    $$ t_x = -(x \cdot \text{eye}) $$
    $$ t_y = -(y \cdot \text{eye}) $$
    $$ t_z = -(z \cdot \text{eye}) $$
    Fill the translation column (column 3):
    $$ \text{out}[12]=t_x, \text{out}[13]=t_y, \text{out}[14]=t_z, \text{out}[15]=1 $$
    Return `out` [S1, unofficial].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import math

def create_identity():
    """Returns a 4x4 identity matrix as Float32Array."""
    return [1.0, 0.0, 0.0, 0.0,
            0.0, 1.0, 0.0, 0.0,
            0.0, 0.0, 1.0, 0.0,
            0.0, 0.0, 0.0, 1.0]

def perspective(out, fov, aspect, near, far):
    """
    Generates a perspective projection matrix.
    out: Float32Array of size 16.
    """
    # Initialize to zero
    for i in range(16):
        out[i] = 0.0
        
    f = 1.0 / math.tan(fov / 2.0)
    
    # Column 0
    out[0] = f / aspect
    # Column 1
    out[5] = f
    # Column 2
    out[10] = (far + near) / (near - far)
    out[11] = -1.0
    # Column 3
    out[14] = (2.0 * far * near) / (near - far)
    
    return out

def look_at(out, eye, center, up):
    """
    Generates a view (look-at) matrix.
    eye, center, up: Lists or arrays of 3 floats [x, y, z].
    """
    # Calculate z-axis (forward)
    z0 = eye[0] - center[0]
    z1 = eye[1] - center[1]
    z2 = eye[2] - center[2]
    
    # Normalize z
    len_z = math.sqrt(z0*z0 + z1*z1 + z2*z2)
    if len_z == 0:
        # Guard against zero-length vector [G]
        z0, z1, z2 = 0.0, 0.0, 1.0
    else:
        inv_len_z = 1.0 / len_z
        z0 *= inv_len_z
        z1 *= inv_len_z
        z2 *= inv_len_z
        
    # Calculate x-axis (right) = cross(up, z)
    x0 = up[1]*z2 - up[2]*z1
    x1 = up[2]*z0 - up[0]*z2
    x2 = up[0]*z1 - up[1]*z0
    
    # Normalize x
    len_x = math.sqrt(x0*x0 + x1*x1 + x2*x2)
    if len_x == 0:
        # Guard against parallel up and view vectors [G]
        x0, x1, x2 = 1.0, 0.0, 0.0
    else:
        inv_len_x = 1.0 / len_x
        x0 *= inv_len_x
        x1 *= inv_len_x
        x2 *= inv_len_x
        
    # Calculate y-axis (up) = cross(z, x)
    y0 = z1*x2 - z2*x1
    y1 = z2*x0 - z0*x2
    y2 = z0*x1 - z1*x0
    
    # Fill rotation columns
    out[0] = x0; out[1] = y0; out[2] = z0; out[3] = 0.0
    out[4] = x1; out[5] = y1; out[6] = z1; out[7] = 0.0
    out[8] = x2; out[9] = y2; out[10] = z2; out[11] = 0.0
    
    # Fill translation column
    out[12] = -(x0*eye[0] + x1*eye[1] + x2*eye[2])
    out[13] = -(y0*eye[0] + y1*eye[1] + y2*eye[2])
    out[14] = -(z0*eye[0] + z1*eye[1] + z2*eye[2])
    out[15] = 1.0
    
    return out
```

## Subtopics

### [Matrix Norms & Utilities](3938-matrix-norms-utilities.md)
Provides fundamental arithmetic operations for 3x3 and 4x4 matrices, including norm calculations, transposition, and multiplication variants. It serves as the computational core for the Custom Matrix Math Library, enabling the construction of view matrices and coordinate transformations without external dependencies. It distinguishes itself by offering explicit implementations for symmetric matrices (`SMat3`) to optimize memory and computation for specific geometric operations like quadratic error function (QEF) accumulation.

### [Scaling Matrix Construction](4126-scaling-matrix-construction.md)
Constructs a 4x4 homogeneous transformation matrix that scales coordinates along the principal axes. It serves as the fundamental building block for non-uniform scaling in 3D graphics, distinct from translation or rotation. It differs from sibling utilities like [Matrix Norms & Utilities](3938-matrix-norms-utilities.md) by performing geometric transformation rather than mathematical analysis.

#### 3D Vector Math Primitives
Includes component-wise multiplication (`operator*(Vector3d v)` returns `(x*v.x, y*v.y, z*v.z)`) and division (`operator/(Vector3d& v)` returns `(x/v.x, y/v.y, z/v.z)`) for 3D vectors, alongside rotation functions such as `rotate_x(theta)` which updates coordinates using $y' = \cos(\theta)y - \sin(\theta)z$ and $z' = \sin(\theta)y + \cos(\theta)z$ after converting degrees to radians [S2, unofficial].

## Operating characteristics
**Time Complexity:** $O(1)$ for matrix generation, as the number of operations is fixed regardless of scene size.
**Memory Complexity:** $O(1)$, requiring only a fixed-size 16-element array per matrix.
**Strengths:** Zero external dependencies; highly optimized for WebGL 2 hardware instancing by producing `Float32Array` outputs directly compatible with GPU buffers [S1, unofficial].
**Limitations:** Lacks advanced features like matrix inversion or decomposition found in general-purpose linear algebra libraries. Precision is limited to standard IEEE 754 single-precision floats.

## Background
Originated in the "CubeWorld-Canyon" project, a WebGL 2 application rendering 250,000 cubes using hardware instancing [S1, unofficial]. The custom library was developed to bypass CPU bottlenecks and external library overhead, allowing direct packing of transformation data into flat arrays for hyper-optimized draw calls [S1, unofficial].

## Sources
[S1] evjapont-hash/CubeWorld-Canyon, https://github.com/evjapont-hash/CubeWorld-Canyon, fetched 2026-10-26 (unofficial)
[S2] JamesRicher/TerminalSignedDistanceFieldRenderer, https://github.com/JamesRicher/TerminalSignedDistanceFieldRenderer, fetched 2026-10-27 (unofficial)
