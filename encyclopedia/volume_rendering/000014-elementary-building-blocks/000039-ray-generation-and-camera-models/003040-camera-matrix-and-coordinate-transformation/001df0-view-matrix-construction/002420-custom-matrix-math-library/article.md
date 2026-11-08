# Custom Matrix Math Library

Also called: Native JavaScript Matrix Operations

## Overview
A custom implementation of 3D transformation primitives (translation, scaling, perspective projection, and look-at) designed to operate natively without external graphics libraries [S1, unofficial]. It solves the dependency problem in environments like JavaScript or embedded C++ by exposing raw arithmetic steps for affine and linear transformations [S2, unofficial][S3]. It differs from high-level graphics libraries by providing explicit, low-level control over matrix construction and vector arithmetic, allowing for custom implementations where standard APIs are unavailable or too heavy [S2, unofficial][S3].

## How it works
**Inputs and Outputs:**
- **Inputs:** Field of view (FOV), aspect ratio, near/far clipping planes, eye position, center point, up vector, and rotation angles.
- **Outputs:** 4x4 transformation matrices (stored as `Float32Array` in JS or equivalent struct in C++) and transformed 3D vectors.

**Data Structures:**
- **Matrix:** A 16-element array representing a 4x4 column-major matrix. In JavaScript, this is a `Float32Array` of length 16. In C++, a struct with 16 `double` or `float` members.
- **Vector:** A 3-element array (`x, y, z`) or struct.

**Numbered Steps:**

1.  **Identity Initialization:** Initialize the output matrix to the identity matrix:
    $$
    I = \begin{bmatrix}
    1 & 0 & 0 & 0 \\
    0 & 1 & 0 & 0 \\
    0 & 0 & 1 & 0 \\
    0 & 0 & 0 & 1
    \end{bmatrix}
    $$
    [S1, unofficial]

2.  **Perspective Projection Construction:**
    - Calculate $f = 1.0 / \tan(\text{fov} / 2)$ [S1, unofficial].
    - Zero out the output matrix.
    - Set elements:
      $$
      \begin{aligned}
      \text{out}[0] &= f / \text{aspect} \\
      \text{out}[5] &= f \\
      \text{out}[10] &= (\text{far} + \text{near}) / (\text{near} - \text{far}) \\
      \text{out}[11] &= -1 \\
      \text{out}[14] &= (2 \times \text{far} \times \text{near}) / (\text{near} - \text{far})
      \end{aligned}
      $$
      [S1, unofficial]

3.  **Look-At Matrix Construction:**
    - Compute the forward vector $z = \text{normalize}(\text{eye} - \text{center})$ [S1, unofficial].
    - Compute the right vector $x = \text{normalize}(\text{cross}(\text{up}, z))$ [S1, unofficial].
    - Compute the up vector $y = \text{cross}(z, x)$ [S1, unofficial].
    - Fill the rotation part of the matrix with columns $x, y, z$:
      $$
      \begin{bmatrix}
      x_0 & y_0 & z_0 & 0 \\
      x_1 & y_1 & z_1 & 0 \\
      x_2 & y_2 & z_2 & 0 \\
      0 & 0 & 0 & 1
      \end{bmatrix}
      $$
      [S1, unofficial]
    - Compute translation components $t_x = -\text{dot}(x, \text{eye})$, $t_y = -\text{dot}(y, \text{eye})$, $t_z = -\text{dot}(z, \text{eye})$ [S1, unofficial].
    - Fill the translation part of the matrix (row 3, columns 0-2):
      $$
      \begin{bmatrix}
      \dots & \dots & \dots & 0 \\
      \dots & \dots & \dots & 0 \\
      \dots & \dots & \dots & 0 \\
      t_x & t_y & t_z & 1
      \end{bmatrix}
      $$
      [S1, unofficial]

4.  **Vector Arithmetic (C++ Variant):**
    - **Component-wise Multiplication:** $v \times w = (v_x w_x, v_y w_y, v_z w_z)$ [S2, unofficial].
    - **Component-wise Division:** $v / w = (v_x / w_x, v_y / w_y, v_z / w_z)$ [S2, unofficial].
    - **Dot Product:** $v \cdot w = v_x w_x + v_y w_y + v_z w_z$ [S2, unofficial].
    - **Rotation around X-axis:**
      $$
      \begin{aligned}
      y' &= \cos(\theta)y - \sin(\theta)z \\
      z' &= \sin(\theta)y + \cos(\theta)z
      \end{aligned}
      $$
      where $\theta$ is converted to radians [S2, unofficial].

**Pseudocode (JavaScript):**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```javascript
// Initialize identity matrix
function createIdentity() {
    return new Float32Array([1,0,0,0, 0,1,0,0, 0,0,1,0, 0,0,0,1]);
}

// Perspective projection matrix
function perspective(out, fov, aspect, near, far) {
    const f = 1.0 / Math.tan(fov / 2);
    out.fill(0);
    out[0] = f / aspect;
    out[5] = f;
    out[10] = (far + near) / (near - far);
    out[11] = -1;
    out[14] = (2 * far * near) / (near - far);
    return out;
}

// Look-at matrix
function lookAt(out, eye, center, up) {
    // Forward vector (z)
    let z0 = eye[0] - center[0];
    let z1 = eye[1] - center[1];
    let z2 = eye[2] - center[2];
    let len = 1 / Math.hypot(z0, z1, z2);
    z0 *= len; z1 *= len; z2 *= len;

    // Right vector (x)
    let x0 = up[1] * z2 - up[2] * z1;
    let x1 = up[2] * z0 - up[0] * z2;
    let x2 = up[0] * z1 - up[1] * z0;
    len = 1 / Math.hypot(x0, x1, x2);
    x0 *= len; x1 *= len; x2 *= len;

    // Up vector (y)
    let y0 = z1 * x2 - z2 * x1;
    let y1 = z2 * x0 - z0 * x2;
    let y2 = z0 * x1 - z1 * x0;

    // Fill matrix (column-major)
    out[0] = x0; out[1] = y0; out[2] = z0; out[3] = 0;
    out[4] = x1; out[5] = y1; out[6] = z1; out[7] = 0;
    out[8] = x2; out[9] = y2; out[10] = z2; out[11] = 0;
    // Translation components calculated as -(dot(axis, eye))
    out[12] = -(x0 * eye[0] + x1 * eye[1] + x2 * eye[2]);
    out[13] = -(y0 * eye[0] + y1 * eye[1] + y2 * eye[2]);
    out[14] = -(z0 * eye[0] + z1 * eye[1] + z2 * eye[2]);
    out[15] = 1;
    return out;
}
```

**Parameters:**
- `fov`: Field of view in radians [S1, unofficial].
- `aspect`: Width/Height ratio [S1, unofficial].
- `near`, `far`: Clipping plane distances [S1, unofficial].
- `eye`, `center`, `up`: Camera position, target, and orientation vector [S1, unofficial].

**Edge Cases and Pitfalls:**
- Division by zero if `near == far` or `aspect == 0` [S1, unofficial].
- Normalization failure if `eye == center` or `up` is parallel to `z` [S1, unofficial].
- Precision loss in very large or very small coordinate systems [G].

## Subtopics

### [Matrix Norms & Utilities](3938-matrix-norms-utilities.md)
Provides fundamental arithmetic operations for 3x3 and 4x4 matrices and vectors, including multiplication, transposition, and norm calculations. It differs from high-level libraries by exposing raw arithmetic steps, enabling custom implementations in JavaScript or embedded C++ without external dependencies.

### [Scaling Matrix Construction](4126-scaling-matrix-construction.md)
Constructs a 4x4 homogeneous transformation matrix for non-uniform scaling along principal axes. It serves as a geometric transformation building block, distinct from translation or rotation, and differs from [Matrix Norms & Utilities](3938-matrix-norms-utilities.md) by performing geometric transformation rather than mathematical analysis.

### 3D Vector Math Primitives
Implements component-wise operations (multiplication, division) and rotations for 3D vectors [S2, unofficial]. Rotations are applied via trigonometric functions (e.g., `rotate_x` uses `cos` and `sin` of the angle in radians) [S2, unofficial].

## Operating characteristics
- **Time Complexity:** Matrix construction is $O(1)$ for fixed-size 4x4 matrices. Vector operations are $O(1)$.
- **Memory Complexity:** $O(1)$ for fixed-size matrices and vectors.
- **Performance:** In JavaScript, using `Float32Array` allows for efficient memory layout and potential GPU transfer [S1, unofficial]. In C++, inline functions minimize overhead [S2, unofficial].
- **Strengths:** No external dependencies, full control over transformation logic, suitable for embedded or web environments [S1, unofficial][S2, unofficial].
- **Limitations:** Lacks optimizations found in specialized libraries (e.g., SIMD, cache-aware layouts) [G].
- **When to use:** When external libraries are unavailable or undesirable, or when custom transformation logic is required [S1, unofficial][S2, unofficial].

## Background
Originated from custom implementations in JavaScript for WebGL 2 rendering [S1, unofficial] and C++ for terminal-based SDF rendering [S2, unofficial]. It is used in environments where lightweight, dependency-free 3D math is needed, such as browser-based visualizations and embedded systems [S1, unofficial][S2, unofficial].

## Sources
[S1] evjapont-hash/CubeWorld-Canyon, https://github.com/evjapont-hash/CubeWorld-Canyon, fetched 2026-10-26 (unofficial)
[S2] JamesRicher/TerminalSignedDistanceFieldRenderer, https://github.com/JamesRicher/TerminalSignedDistanceFieldRenderer, fetched 2026-10-27 (unofficial)
