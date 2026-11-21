# Jacobian Computation for Coordinate Transformation
## Overview
Computes a 3×3 Jacobian matrix for a batch of 3D points to facilitate coordinate space transformations or derivative propagation in rendering pipelines [S1, unofficial]. It differs from general numerical Jacobian estimation by using a closed-form analytical derivation based on the input vector's components and Euclidean norm, rather than finite differences [S1, unofficial].

## How it works
**Inputs:** A tensor `a` of shape $(N, 3)$ containing $N$ 3D points $(a_x, a_y, a_z)$ [S1, unofficial].
**Outputs:** A tensor `J` of shape $(N, 9)$ containing the Jacobian elements in row-major order, or $(N, 3, 3)$ if reshaped [S1, unofficial].

**Data Structures:**
- `a`: Float tensor, shape $(N, 3)$ [S1, unofficial].
- `r`: Float tensor, shape $(N,)$, representing the inverse Euclidean norm [S1, unofficial].
- `J`: Float tensor, shape $(N, 9)$ or $(N, 3, 3)$ [S1, unofficial].

**Parameters:**
- Not given in the sources.

**Steps:**
1. Extract components $a_x, a_y, a_z$ from `a` [S1, unofficial].
2. Compute the inverse Euclidean norm: $r = 1 / \sqrt{a_x^2 + a_y^2 + a_z^2}$ [S1, unofficial].
3. Construct the 9 elements of the Jacobian matrix $J$ according to the formula below [S1, unofficial].

**Formula:**
The Jacobian matrix $J$ is defined as:
$$
J = \begin{bmatrix}
\frac{1}{a_z} & 0 & -\frac{a_x}{a_z^2} \\
0 & \frac{1}{a_z} & -\frac{a_y}{a_z^2} \\
a_x \cdot r & a_y \cdot r & a_z \cdot r
\end{bmatrix}
$$
where $r = \frac{1}{\sqrt{a_x^2 + a_y^2 + a_z^2}}$ [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import torch

def compute_jacobian(a: torch.Tensor) -> torch.Tensor:
    """
    Computes the Jacobian matrix for coordinate transformation.
    
    Args:
        a: Tensor of shape (N, 3) with coordinates (x, y, z).
        
    Returns:
        Tensor of shape (N, 9) with Jacobian elements in row-major order.
    """
    # Unpack coordinates
    ax = a[:, 0]
    ay = a[:, 1]
    az = a[:, 2]
    
    # Compute inverse Euclidean norm r = 1 / sqrt(x^2 + y^2 + z^2)
    r = torch.rsqrt(ax**2 + ay**2 + az**2)
    
    # Compute terms involving az
    # Note: Division by zero if az == 0 results in Inf/NaN
    inv_az = 1.0 / az
    inv_az_sq = inv_az * inv_az
    
    # Construct the 9 elements in row-major order
    # Row 0
    j00 = inv_az
    j01 = torch.zeros_like(ax)
    j02 = -ax * inv_az_sq
    
    # Row 1
    j10 = torch.zeros_like(ax)
    j11 = inv_az
    j12 = -ay * inv_az_sq
    
    # Row 2
    j20 = ax * r
    j21 = ay * r
    j22 = az * r
    
    # Stack into (N, 9)
    J = torch.stack([j00, j01, j02,
                     j10, j11, j12,
                     j20, j21, j22], dim=1)
    
    return J
```

**Edge Cases and Pitfalls:**
- **Division by Zero:** If any point has $a_z = 0$, the terms $1/a_z$ and $1/a_z^2$ will produce `Inf` or `NaN` [S1, unofficial]. The source code does not explicitly clamp or mask these cases [S1, unofficial]. Implementers must ensure $a_z \neq 0$ or add a small epsilon to the denominator [G].

## Subtopics

### VF-NeRF Directional Derivative Computation
Computes directional derivatives of a neural network's output with respect to input coordinates using the Jacobian [S2, unofficial]. It constructs two orthogonal normal vectors $n_1 = (n_y, -n_x, 0)$ and $n_2 = \text{cross}(n, n_1)$ relative to a primary normal $n$, then multiplies the Jacobian by these normalized vectors to get directional derivatives [S2, unofficial]. It also provides a numerical baseline using central finite differences with $\epsilon = 10^{-5}$, computing $J[:, i] = (\text{network}(pos_h)[:, :3] - \text{network}(neg_h)[:, :3]) / (2\epsilon)$ where $pos_h$ and $neg_h$ are points perturbed by $\pm \epsilon$ along axis $i$ [S2, unofficial].

## Operating characteristics
- **Time Complexity:** $O(N)$ for $N$ points, involving basic arithmetic operations [G].
- **Memory Complexity:** $O(N)$ to store the output Jacobian tensor [G].
- **Strengths:** Analytical computation is significantly faster than numerical finite differences [S2, unofficial].
- **Limitations:** Singularities at $a_z = 0$ require external handling [S1, unofficial]. Not suitable for general coordinate systems where $a_z$ might be zero [G].

## Background
This method is used to transform derivatives between coordinate spaces, likely for gradient propagation during optimization [S1, unofficial]. It appears in 3D Gaussian Splatting implementations [S1, unofficial].

## Sources
[S1] https://github.com/WangFeng18/3d-gaussian-splatting, https://github.com/WangFeng18/3d-gaussian-splatting, fetched 2026-10-30 (unofficial)
[S2] albertgassol1/vf-nerf, https://github.com/albertgassol1/vf-nerf, fetched 2026-10-26 (unofficial)
