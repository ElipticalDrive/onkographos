# SE2 Lie Group Implementation

Also called: SE2, Special Euclidean Group 2D

## Overview
The SE2 Lie group implementation parameterizes 2D rigid body transformations using a tangent parameterization of $(v_x, v_y, \omega)$ and an internal parameterization of $(\cos, \sin, x, y)$ [S1, unofficial]. It operates in 2D rather than 3D like [Quaternion Rotation](2302-quaternion-rotation.md), and includes translation components unlike [2D Rotation](3357-2d-rotation.md) [G].

## How it works
**Inputs and Outputs:**
The method takes standard 2D pose parameters $(x, y, \theta)$ or tangent vectors $(v_x, v_y, \omega)$ and outputs an SE2 transformation object, which can be converted to a $3 \times 3$ homogeneous matrix [S1, unofficial].

**Data Structures:**
The `SE2` class stores internal parameters in the field `unit_complex_xy` of type `onpt.NDArray[onp.floating]` with shape containing at least 4 elements, representing $(\cos, \sin, x, y)$ [S1, unofficial].

**Steps:**

1.  **Construction from Pose $(x, y, \theta)$:**
    Compute $c = \cos(\theta)$ and $s = \sin(\theta)$ [S1, unofficial]. Construct the array $[c, s, x, y]$ and return `SE2(unit_complex_xy=array)` [S1, unofficial].

2.  **Construction from SO2 and Translation:**
    Given an SO2 rotation object with `unit_complex` $[c, s]$ and a 2D translation vector $[t_x, t_y]$, concatenate them into $[c, s, t_x, t_y]$ and return `SE2(unit_complex_xy=array)` [S1, unofficial].

3.  **Conversion to Homogeneous Matrix:**
    Unpack `unit_complex_xy` into $c, s, x, y$ [S1, unofficial]. Return the matrix:
    $$
    M = \begin{bmatrix}
    c & -s & x \\
    s & c & y \\
    0.0 & 0.0 & 1.0
    \end{bmatrix}
    $$ [S1, unofficial].

4.  **Exponential Map $\exp(\xi)$:**
    Input: Tangent vector $\xi = [v_x, v_y, \theta]$. Let $\epsilon$ be a small threshold. If $|\theta| < \epsilon$, compute Taylor approximations: $\text{sin\_over\_theta} = 1.0 - \theta^2/6.0$ and $\text{one\_minus\_cos\_over\_theta} = 0.5\theta - \theta^3/24.0$ [S1, unofficial]. Otherwise, compute $\text{sin\_over\_theta} = \sin(\theta)/\theta$ and $\text{one\_minus\_cos\_over\_theta} = (1 - \cos(\theta))/\theta$ [S1, unofficial]. Construct matrix $V$:
    $$
    V = \begin{bmatrix}
    \text{sin\_over\_theta} & \text{one\_minus\_cos\_over\_theta} \\
    -\text{one\_minus\_cos\_over\_theta} & \text{sin\_over\_theta}
    \end{bmatrix}
    $$ [S1, unofficial]. Compute rotation $R = \text{SO2}(\theta)$ and translation $t = V \cdot [v_x, v_y]^T$. Return `SE2.from_rotation_and_translation(rotation=R, translation=t)` [S1, unofficial].

5.  **Logarithm Map $\log(g)$:**
    Input: SE2 transformation $g$. Extract $\theta$ from the rotation component. Compute $c_{-1} = \cos(\theta) - 1.0$ [S1, unofficial]. If $|c_{-1}| < \epsilon$, compute Taylor approximation $\text{half\_theta\_over\_tan\_half\_theta} = 1.0 - \theta^2/12.0$ [S1, unofficial]. Otherwise, compute $\text{half\_theta\_over\_tan\_half\_theta} = -(\theta/2 \cdot \sin(\theta)) / c_{-1}$ [S1, unofficial]. Construct $V_{inv}$:
    $$
    V_{inv} = \begin{bmatrix}
    \text{half\_theta\_over\_tan\_half\_theta} & -\theta/2 \\
    \theta/2 & \text{half\_theta\_over\_tan\_half\_theta}
    \end{bmatrix}
    $$ [S1, unofficial]. Compute $t_{xy} = V_{inv} \cdot \text{translation}$. Return tangent vector $[t_{xy}[0], t_{xy}[1], \theta]$ [S1, unofficial].

6.  **Adjoint Matrix $\text{Ad}_g$:**
    Unpack `unit_complex_xy` into $c, s, x, y$ [S1, unofficial]. Return:
    $$
    \text{Ad}_g = \begin{bmatrix}
    c & -s & y \\
    s & c & -x \\
    0.0 & 0.0 & 1.0
    \end{bmatrix}
    $$ [S1, unofficial].

**Parameters:**
- $\epsilon$: Threshold for Taylor approximation switch. Value not given in the sources [S1, unofficial].

**Edge Cases and Pitfalls:**
- **NaNs in AD:** When using JAX reverse-mode automatic differentiation with `onp.where`, branches taken for Taylor approximations can cause NaNs in non-Taylor branches during gradient computation. To fix this, use a "shim" value of 1.0 for `safe_theta` and `safe_cos_minus_one` when the Taylor branch is taken, ensuring non-zero denominators in the non-Taylor branches [S1, unofficial].
- **Small Angles:** Direct division by $\theta$ or $\cos(\theta)-1$ causes numerical instability for small angles. Taylor approximations must be used [S1, unofficial].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import numpy as np

def se2_exp(tangent):
    """
    Compute the exponential map for SE2.
    tangent: array-like [vx, vy, theta]
    Returns: SE2 object
    """
    vx, vy, theta = tangent[0], tangent[1], tangent[2]
    
    # Safe theta for AD to prevent NaNs in non-Taylor branches
    safe_theta = np.where(np.abs(theta) < 1e-6, 1.0, theta)
    
    if np.abs(theta) < 1e-6:
        sin_over_theta = 1.0 - theta**2 / 6.0
        one_minus_cos_over_theta = 0.5 * theta - theta**3 / 24.0
    else:
        sin_over_theta = np.sin(theta) / safe_theta
        one_minus_cos_over_theta = (1.0 - np.cos(theta)) / safe_theta
        
    V = np.array([
        [sin_over_theta, one_minus_cos_over_theta],
        [-one_minus_cos_over_theta, sin_over_theta]
    ])
    
    translation = V @ np.array([vx, vy])
    rotation = SO2.from_radians(theta)
    
    return SE2.from_rotation_and_translation(rotation, translation)

def se2_log(se2_obj):
    """
    Compute the logarithm map for SE2.
    se2_obj: SE2 object
    Returns: array [vx, vy, theta]
    """
    theta = se2_obj.rotation.theta
    translation = se2_obj.translation
    
    cos_minus_one = np.cos(theta) - 1.0
    safe_cos_minus_one = np.where(np.abs(cos_minus_one) < 1e-6, 1.0, cos_minus_one)
    half_theta = theta / 2.0
    
    if np.abs(cos_minus_one) < 1e-6:
        half_theta_over_tan_half_theta = 1.0 - theta**2 / 12.0
    else:
        half_theta_over_tan_half_theta = -(half_theta * np.sin(theta)) / safe_cos_minus_one
        
    V_inv = np.array([
        [half_theta_over_tan_half_theta, -half_theta],
        [half_theta, half_theta_over_tan_half_theta]
    ])
    
    t_xy = V_inv @ translation
    return np.concatenate([t_xy, [theta]])

def se2_to_matrix(se2_obj):
    """
    Convert SE2 to 3x3 homogeneous matrix.
    """
    c, s, x, y = se2_obj.unit_complex_xy
    return np.array([
        [c, -s, x],
        [s, c, y],
        [0.0, 0.0, 1.0]
    ])

def se2_adjoint(se2_obj):
    """
    Compute the adjoint matrix for SE2.
    """
    c, s, x, y = se2_obj.unit_complex_xy
    return np.array([
        [c, -s, y],
        [s, c, -x],
        [0.0, 0.0, 1.0]
    ])

class SE2:
    def __init__(self, unit_complex_xy):
        self.unit_complex_xy = np.array(unit_complex_xy)
        
    @staticmethod
    def from_pose(x, y, theta):
        c = np.cos(theta)
        s = np.sin(theta)
        return SE2([c, s, x, y])
        
    @staticmethod
    def from_rotation_and_translation(rotation, translation):
        c, s = rotation.unit_complex
        return SE2([c, s, translation[0], translation[1]])
        
    @property
    def rotation(self):
        c, s = self.unit_complex_xy[0], self.unit_complex_xy[1]
        return SO2([c, s])
        
    @property
    def translation(self):
        return self.unit_complex_xy[2:4]

class SO2:
    def __init__(self, unit_complex):
        self.unit_complex = np.array(unit_complex)
        
    @property
    def theta(self):
        return np.arctan2(self.unit_complex[1], self.unit_complex[0])
        
    @staticmethod
    def from_radians(theta):
        return SO2([np.cos(theta), np.sin(theta)])
```

## Operating characteristics
- **Time and memory complexity:** Not given in the sources [S1, unofficial].
- **Measured performance:** Not given in the sources [S1, unofficial].
- **Strengths:** Compact storage of 4 floats; differentiable via automatic differentiation with proper AD guards [S1, unofficial].
- **Limitations:** Only applicable to 2D rigid motions; requires careful handling of numerical stability for small angles to avoid NaNs in gradients [S1, unofficial].
- **When to use:** Optimizing 2D camera poses or rigid body transformations in 2D space [G].
- **When not to use:** For 3D transformations or pure rotation without translation [G].

## Background
- **Origin and history:** Not given in the sources [S1, unofficial].
- **Usage:** Not given in the sources [S1, unofficial].

## Sources
[S1] https://github.com/WangFeng18/3d-gaussian-splatting, https://github.com/WangFeng18/3d-gaussian-splatting, fetched 2026-10-30 (unofficial)
