# Custom Matrix Math Library

Also called: Native JavaScript Matrix Operations

## Overview
A custom implementation of 3D transformation mathematics (translation, scaling, perspective projection, and look-at) written natively in JavaScript to eliminate dependencies on external linear algebra libraries [S1, unofficial]. It distinguishes itself from sibling methods by providing a self-contained, lightweight API for constructing camera matrices and handling vector arithmetic directly within the browser environment, specifically optimized for WebGL 2 hardware instancing pipelines [S1, unofficial].

## How it works
**Inputs and Outputs:**
- **Inputs:** Camera parameters (eye position, center point, up vector, field of view, aspect ratio, near/far planes) [S1, unofficial] and vector components [S2, unofficial].
- **Outputs:** A 16-element array representing a 4x4 matrix, or transformed 3D vectors.

**Data Structures:**
- **Matrix:** A 16-element array storing the 4x4 matrix. The source code populates it in row-major order: indices 0-3 hold the first row (x-axis components and 0), indices 4-7 hold the second row (y-axis components and 0), indices 8-11 hold the third row (z-axis components and 0), and indices 12-15 hold the translation and homogeneous components [S1, unofficial].
- **Vector:** A 3-component structure with fields `x`, `y`, `z` [S2, unofficial].

**Steps:**

1. **Perspective Projection Matrix Construction:** Calculate the perspective matrix elements using the field of view (FOV), aspect ratio, and near/far clipping planes [S1, unofficial].
   - Compute $f = 1.0 / \tan(\text{fov} / 2)$ [S1, unofficial].
   - Initialize output array to zeros.
   - Set `out[0] = f / aspect` [S1, unofficial].
   - Set `out[5] = f` [S1, unofficial].
   - Set `out[10] = (\text{far} + \text{near}) / (\text{near} - \text{far})` [S1, unofficial].
   - Set `out[11] = -1` [S1, unofficial].
   - Set `out[14] = (2 * \text{far} * \text{near}) / (\text{near} - \text{far})` [S1, unofficial].
   - Return `out` [S1, unofficial].

2. **LookAt Matrix Construction:** Construct the view matrix from eye, center, and up vectors [S1, unofficial].
   - Compute the forward vector $z = \text{normalize}(\text{eye} - \text{center})$ [S1, unofficial].
     - $z_0 = \text{eye}_x - \text{center}_x$, etc. [S1, unofficial].
     - $len = 1 / \sqrt{z_0^2 + z_1^2 + z_2^2}$ [S1, unofficial].
     - $z_0 *= len, z_1 *= len, z_2 *= len$ [S1, unofficial].
   - Compute the right vector $x = \text{normalize}(\text{up} \times z)$ [S1, unofficial].
     - $x_0 = \text{up}_y z_2 - \text{up}_z z_1$ [S1, unofficial].
     - $x_1 = \text{up}_z z_0 - \text{up}_x z_2$ [S1, unofficial].
     - $x_2 = \text{up}_x z_1 - \text{up}_y z_0$ [S1, unofficial].
     - $len = 1 / \sqrt{x_0^2 + x_1^2 + x_2^2}$ [S1, unofficial].
     - $x_0 *= len, x_1 *= len, x_2 *= len$ [S1, unofficial].
   - Compute the up vector $y = z \times x$ [S1, unofficial].
     - $y_0 = z_1 x_2 - z_2 x_1$ [S1, unofficial].
     - $y_1 = z_2 x_0 - z_0 x_2$ [S1, unofficial].
     - $y_2 = z_0 x_1 - z_1 x_0$ [S1, unofficial].
   - Fill the matrix rows:
     - `out[0]=x0, out[1]=y0, out[2]=z0, out[3]=0` [S1, unofficial].
     - `out[4]=x1, out[5]=y1, out[6]=z1, out[7]=0` [S1, unofficial].
     - `out[8]=x2, out[9]=y2, out[10]=z2, out[11]=0` [S1, unofficial].
     - Translation components `out[12..15]` are set to $-(x \cdot \text{eye}), -(y \cdot \text{eye}), -(z \cdot \text{eye}), 1$ [S1, unofficial].

3. **Vector Arithmetic:**
   - **Component-wise Multiplication:** `v * w` returns $(v_x w_x, v_y w_y, v_z w_z)$ [S2, unofficial].
   - **Component-wise Division:** `v / w` returns $(v_x / w_x, v_y / w_y, v_z / w_z)$ [S2, unofficial].
   - **Rotation:** Rotating a vector around the X-axis by angle $\theta$ (converted to radians):
     - $y' = \cos(\theta)y - \sin(\theta)z$
     - $z' = \sin(\theta)y + \cos(\theta)z$
     - $x' = x$ [S2, unofficial].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import math

def create_identity_matrix():
    """Returns a 4x4 identity matrix as a list of 16 floats."""
    return [1.0, 0.0, 0.0, 0.0,
            0.0, 1.0, 0.0, 0.0,
            0.0, 0.0, 1.0, 0.0,
            0.0, 0.0, 0.0, 1.0]

def perspective_matrix(fov, aspect, near, far):
    """
    Constructs a perspective projection matrix.
    fov: Field of view in radians.
    aspect: Width / Height.
    near: Near clipping plane distance.
    far: Far clipping plane distance.
    Returns: List of 16 floats (row-major).
    """
    f = 1.0 / math.tan(fov / 2.0)
    out = [0.0] * 16
    out[0] = f / aspect
    out[5] = f
    out[10] = (far + near) / (near - far)
    out[11] = -1.0
    out[14] = (2.0 * far * near) / (near - far)
    return out

def look_at_matrix(eye, center, up):
    """
    Constructs a view matrix.
    eye: [ex, ey, ez]
    center: [cx, cy, cz]
    up: [ux, uy, uz]
    Returns: List of 16 floats (row-major).
    """
    # Forward vector (z)
    z0 = eye[0] - center[0]
    z1 = eye[1] - center[1]
    z2 = eye[2] - center[2]
    len_z = 1.0 / math.sqrt(z0*z0 + z1*z1 + z2*z2)
    z0 *= len_z
    z1 *= len_z
    z2 *= len_z

    # Right vector (x)
    x0 = up[1] * z2 - up[2] * z1
    x1 = up[2] * z0 - up[0] * z2
    x2 = up[0] * z1 - up[1] * z0
    len_x = 1.0 / math.sqrt(x0*x0 + x1*x1 + x2*x2)
    x0 *= len_x
    x1 *= len_x
    x2 *= len_x

    # Up vector (y)
    y0 = z1 * x2 - z2 * x1
    y1 = z2 * x0 - z0 * x2
    y2 = z0 * x1 - z1 * x0

    # Fill matrix (row-major)
    out = [0.0] * 16
    out[0] = x0; out[1] = y0; out[2] = z0; out[3] = 0.0
    out[4] = x1; out[5] = y1; out[6] = z1; out[7] = 0.0
    out[8] = x2; out[9] = y2; out[10] = z2; out[11] = 0.0
    
    # Translation components: -(basis * eye)
    out[12] = -(x0 * eye[0] + x1 * eye[1] + x2 * eye[2])
    out[13] = -(y0 * eye[0] + y1 * eye[1] + y2 * eye[2])
    out[14] = -(z0 * eye[0] + z1 * eye[1] + z2 * eye[2])
    out[15] = 1.0
    
    return out

def rotate_x_vector(v, theta_deg):
    """
    Rotates a 3D vector around the X-axis.
    v: [x, y, z]
    theta_deg: Angle in degrees.
    Returns: [x', y', z']
    """
    theta = theta_deg * (math.pi / 180.0) # DEG_TO_RAD conversion
    cos_t = math.cos(theta)
    sin_t = math.sin(theta)
    
    x_new = v[0]
    y_new = cos_t * v[1] - sin_t * v[2]
    z_new = sin_t * v[1] + cos_t * v[2]
    
    return [x_new, y_new, z_new]
```

**Parameters:**
- `fov`: Field of view in radians. [S1, unofficial].
- `aspect`: Screen aspect ratio (width/height). [S1, unofficial].
- `near`, `far`: Clipping plane distances. [S1, unofficial].

**Edge Cases and Pitfalls:**
- The implementation computes the inverse length directly without explicit guards for degenerate inputs (e.g., `eye == center` or `up` parallel to `z`), which will produce NaN or infinity values in those cases [S1, unofficial].

## Subtopics

### [Matrix Norms & Utilities](3938-matrix-norms-utilities.md)
Provides fundamental arithmetic operations, norms, and data structures for 3x3 and 4x4 matrices used in volume rendering transformations. It distinguishes itself by offering specialized handling for symmetric matrices (via `SMat3`) to optimize storage and computation for covariance-like operations, alongside standard Frobenius and off-diagonal norms for error analysis.

### [Scaling Matrix Construction](4126-scaling-matrix-construction.md)
Constructs a 4x4 diagonal matrix that scales coordinates along the X, Y, and Z axes. It solves the problem of applying non-uniform or uniform scaling transformations to 3D geometry within a homogeneous coordinate system. It is distinguished from [Scaling and Rotation Matrix Construction](7986-scaling-and-rotation-matrix-construction.md) by excluding rotational degrees of freedom, and from [Transformation Matrix Construction](12879-transformation-matrix-construction.md) by excluding translation.

## Operating characteristics
- **Time Complexity:** Matrix construction is $O(1)$ for fixed-size 4x4 matrices. Vector operations are $O(1)$.
- **Memory Complexity:** $O(1)$ space, using fixed-size arrays (16 floats for matrices, 3 floats for vectors).
- **Performance:** Optimized for WebGL 2 hardware instancing, allowing rendering of 250,000 cubes at a high framerate via a single draw call by packing data into `Float32Arrays` [S1, unofficial].
- **Strengths:** No external dependencies, lightweight, native JavaScript implementation suitable for browser-based volume rendering.
- **Limitations:** Lacks advanced features found in full linear algebra libraries (e.g., decomposition, inversion algorithms beyond basic lookAt). Precision is limited to standard IEEE 754 double precision.

## Background
Originated in the "CubeWorld-Canyon" project to handle 3D transformations natively in JavaScript without external libraries, supporting WebGL 2 hardware instancing of 250,000 cubes [S1, unofficial].

## Sources
[S1] evjapont-hash/CubeWorld-Canyon, https://github.com/evjapont-hash/CubeWorld-Canyon, fetched 2026-10-26 (unofficial)
[S2] JamesRicher/TerminalSignedDistanceFieldRenderer, https://github.com/JamesRicher/TerminalSignedDistanceFieldRenderer, fetched 2026-10-27 (unofficial)
