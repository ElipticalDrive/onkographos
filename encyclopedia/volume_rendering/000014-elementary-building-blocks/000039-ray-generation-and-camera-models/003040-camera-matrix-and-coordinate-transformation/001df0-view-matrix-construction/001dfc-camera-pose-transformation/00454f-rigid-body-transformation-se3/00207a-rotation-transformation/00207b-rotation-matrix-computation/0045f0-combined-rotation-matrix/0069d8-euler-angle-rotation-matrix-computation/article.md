# Euler Angle Rotation Matrix Computation

Also called: Rotated unit vectors calculation, Z-Y-X Euler Rotation Matrix

## Overview
This method computes the $3 \times 3$ rotation matrix corresponding to a sequence of rotations about the Z, Y, and X axes (Z-Y-X Euler angles) to transform unit vectors from object space to display space [S1, unofficial]. It is specifically designed for volume rendering ray generation, where the resulting matrix components are stored as global variables representing the unit steps along the transformed axes [S1, unofficial]. It differs from general-purpose rotation libraries by targeting a specific fixed-point or integer-based input pipeline and storing results in a flat global variable namespace rather than a structured matrix object [S1, unofficial].

## How it works
**Inputs:**
- `X.rot.degr`: Rotation angle around the X-axis in degrees.
- `Y.rot.degr`: Rotation angle around the Y-axis in degrees.
- `Z.rot.degr`: Rotation angle around the Z-axis in degrees.

**Outputs:**
- Global variables `unit.step.xx` through `unit.step.zz` (9 values), representing the elements of the rotation matrix $R$. Specifically, `unit.step.ab` corresponds to $R_{ab}$ where $a$ is the row and $b$ is the column.

**Data Structures:**
- The source uses a language with explicit type declarations (`REAL32`) and global variables. The output is not a contiguous array but nine distinct global identifiers.
- Constants: `DR180` is defined as `180.0` (REAL32). `PI` is the mathematical constant $\pi$.

**Steps:**
1.  **Degree to Radian Conversion:** Convert each input angle from degrees to radians. The source specifies rounding the input degree value to the nearest integer before conversion [S1, unofficial].
    $$ \theta_{rad} = \frac{\text{ROUND}(\theta_{deg}) \times \pi}{180.0} $$
    Where $\theta \in \{X, Y, Z\}$.

2.  **Trigonometric Pre-computation:** Calculate the sine and cosine for each axis's radian angle [S1, unofficial].
    $$ S_X = \sin(X_{rad}), \quad C_X = \cos(X_{rad}) $$
    $$ S_Y = \sin(Y_{rad}), \quad C_Y = \cos(Y_{rad}) $$
    $$ S_Z = \sin(Z_{rad}), \quad C_Z = \cos(Z_{rad}) $$

3.  **Matrix Element Calculation:** Compute the 9 elements of the rotation matrix using the Z-Y-X intrinsic rotation order (rotate about Z, then Y, then X). The matrix $R$ is the product $R_X(R_X) R_Y(R_Y) R_Z(R_Z)$ [S1, unofficial].

**Formulas:**
The rotation matrix $R$ is defined as:
$$
R = \begin{bmatrix}
R_{xx} & R_{xy} & R_{xz} \\
R_{yx} & R_{yy} & R_{yz} \\
R_{zx} & R_{zy} & R_{zz}
\end{bmatrix}
$$

The elements are calculated as follows [S1, unofficial]:
$$
\begin{aligned}
R_{xx} &= C_Y C_Z \\
R_{xy} &= C_Y S_Z \\
R_{xz} &= -S_Y \\
R_{yx} &= S_X S_Y C_Z - C_X S_Z \\
R_{yy} &= S_X S_Y S_Z + C_X C_Z \\
R_{yz} &= S_X C_Y \\
R_{zx} &= C_X S_Y C_Z + S_X S_Z \\
R_{zy} &= C_X S_Y S_Z - S_X C_Z \\
R_{zz} &= C_X C_Y
\end{aligned}
$$

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def compute_euler_rotation_matrix(x_deg, y_deg, z_deg):
    """
    Computes the Z-Y-X Euler rotation matrix elements.
    Inputs are in degrees. Outputs are the 9 matrix elements.
    """
    # Constants
    DR180 = 180.0
    PI = 3.14159265358979323846

    # Step 1: Convert degrees to radians
    # Note: Source [S1, unofficial] specifies rounding the degree input to integer first.
    x_rad = ((round(x_deg) * PI) / DR180)
    y_rad = ((round(y_deg) * PI) / DR180)
    z_rad = ((round(z_deg) * PI) / DR180)

    # Step 2: Calculate sine and cosine factors
    sin_x = sin(x_rad)
    cos_x = cos(x_rad)
    sin_y = sin(y_rad)
    cos_y = cos(y_rad)
    sin_z = sin(z_rad)
    cos_z = cos(z_rad)

    # Step 3: Compute unit steps (matrix elements)
    # Mapping: unit.step.ab -> R_ab
    unit_step_xx = cos_y * cos_z
    unit_step_xy = cos_y * sin_z
    unit_step_xz = -sin_y
    
    unit_step_yx = (sin_x * sin_y * cos_z) - (cos_x * sin_z)
    unit_step_yy = (sin_x * sin_y * sin_z) + (cos_x * cos_z)
    unit_step_yz = sin_x * cos_y
    
    unit_step_zx = (cos_x * sin_y * cos_z) + (sin_x * sin_z)
    unit_step_zy = (cos_x * sin_y * sin_z) - (sin_x * cos_z)
    unit_step_zz = cos_x * cos_y

    return {
        'xx': unit_step_xx, 'xy': unit_step_xy, 'xz': unit_step_xz,
        'yx': unit_step_yx, 'yy': unit_step_yy, 'yz': unit_step_yz,
        'zx': unit_step_zx, 'zy': unit_step_zy, 'zz': unit_step_zz
    }
```

**Edge Cases and Pitfalls:**
- **Rounding:** The source explicitly rounds the input degree values to the nearest integer before conversion [S1, unofficial]. This suggests the implementation targets systems where floating-point precision might be limited or where inputs are expected to be integer degrees.
- **Global State:** The original implementation stores results in global variables (`unit.step.*`) [S1, unofficial]. In a modern implementation, these should be returned as a structure or array to avoid side effects.
- **Order of Operations:** The formulas correspond to a Z-Y-X rotation sequence (intrinsic rotations) or X-Y-Z extrinsic rotations. Deviating from this order changes the matrix entirely.

## Operating characteristics
- **Time Complexity:** $O(1)$. The computation involves a fixed number of trigonometric functions and arithmetic operations.
- **Memory Complexity:** $O(1)$. Only a few temporary variables and the 9 output values are required.
- **Strengths:** Simple to implement; directly provides the basis vectors (unit steps) needed for ray marching in volume rendering [S1, unofficial].
- **Limitations:** The specific rounding behavior may introduce small quantization errors compared to direct float-to-radian conversion. The global variable storage pattern is not thread-safe.
- **When to use:** When implementing legacy volume rendering pipelines or specific hardware shaders that expect pre-computed unit vectors for ray direction calculation [S1, unofficial].

## Background
This method originates from early voxel data processing systems running on Transputer networks [S1, unofficial]. It was designed to efficiently transform object-space unit vectors into display space for ray casting, a fundamental step in volume rendering [S1, unofficial]. The specific variable naming convention (`unit.step.*`) and the degree-to-radian conversion with rounding reflect the implementation constraints and style of the era (early 1990s) [S1, unofficial].

## Sources
[S1] DTIC ADA245410: Voxel Data Processing on a Transputer Network, https://archive.org/details/DTIC_ADA245410, fetched 2026-10-26 (unofficial)
