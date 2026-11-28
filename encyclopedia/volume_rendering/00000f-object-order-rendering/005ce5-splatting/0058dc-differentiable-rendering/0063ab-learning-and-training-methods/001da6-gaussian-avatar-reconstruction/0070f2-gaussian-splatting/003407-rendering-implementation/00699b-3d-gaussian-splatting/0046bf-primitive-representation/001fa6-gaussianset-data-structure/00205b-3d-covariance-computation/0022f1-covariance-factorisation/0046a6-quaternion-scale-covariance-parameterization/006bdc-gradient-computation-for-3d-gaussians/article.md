# Gradient Computation for 3D Gaussians

Also called: Chain rule gradient for scale and rotation, Gradient of 3D Gaussian parameters

## Overview
This method computes the gradients of the rendering loss with respect to the rotation quaternion $q$ and scaling vector $s$ of 3D Gaussians. It solves the problem of backpropagating errors through the non-linear parameterization of the covariance matrix $\Sigma$, which is defined as $\Sigma = RSS^T R^T$ where $R$ is the rotation matrix derived from $q$ and $S$ is the diagonal scaling matrix derived from $s$. It sets itself apart by explicitly deriving the partial derivatives of the intermediate matrix $M=RS$ with respect to the quaternion components and scale components, allowing for efficient, closed-form gradient computation without numerical differentiation [S1].

## How it works
**Inputs and Outputs:**
*   **Inputs:**
    *   $q$: Unit quaternion with real part $q_r$ and imaginary parts $q_i, q_j, q_k$ [S1].
    *   $s$: Scaling vector with components $s_X, s_Y, s_Z$ [S1].
    *   $\frac{\partial Z}{\partial M}$: Gradient of the loss $Z$ with respect to the intermediate matrix $M$ [S1].
*   **Outputs:**
    *   $\frac{\partial Z}{\partial q}$: Gradient of the loss with respect to the quaternion components.
    *   $\frac{\partial Z}{\partial s}$: Gradient of the loss with respect to the scale components.

**Data Structures:**
*   $q$: A 4-element vector $[q_r, q_i, q_j, q_k]$ [S1].
*   $s$: A 3-element vector $[s_X, s_Y, s_Z]$ [S1].
*   $M$: A $3 \times 3$ matrix defined as $M = RS$ [S1].
*   $R(q)$: A $3 \times 3$ rotation matrix derived from $q$ [S1].

**Mathematical Derivation:**
The covariance matrix is defined as $\Sigma = MM^T$ where $M = RS$. The gradient of the loss $Z$ with respect to $M$ is given by the chain rule:
$$ \frac{\partial Z}{\partial M} = 2M^T \frac{\partial Z}{\partial \Sigma} $$
Since $\Sigma$ is symmetric, this compact form is used [S1]. The gradients for $s$ and $q$ are then computed via:
$$ \frac{\partial Z}{\partial s} = \frac{\partial Z}{\partial M} \frac{\partial M}{\partial s} $$
$$ \frac{\partial Z}{\partial q} = \frac{\partial Z}{\partial M} \frac{\partial M}{\partial q} $$
Note: The source notation $\frac{d\Sigma'}{ds}$ refers to the projected covariance $\Sigma'$, but the core gradient derivation for the parameters $s$ and $q$ relies on the world/view space covariance $\Sigma$ and the intermediate matrix $M$ [S1].

**Rotation Matrix $R(q)$:**
The rotation matrix is computed from the unit quaternion $q$ as [S1]:
$$ R(q) = 2 \begin{pmatrix} \frac{1}{2} - (q_j^2 + q_k^2) & (q_i q_j - q_r q_k) & (q_i q_k + q_r q_j) \\ (q_i q_j + q_r q_k) & \frac{1}{2} - (q_i^2 + q_k^2) & (q_j q_k - q_r q_i) \\ (q_i q_k - q_r q_j) & (q_j q_k + q_r q_i) & \frac{1}{2} - (q_i^2 + q_k^2) \end{pmatrix} $$
*Correction Note:* The source text contains a typo in the $(2,3)$ and $(3,2)$ entries of the matrix provided in the notes vs the standard derivation. The source text explicitly lists:
Row 2: $(q_i q_j + q_r q_k)$, $(\frac{1}{2} - (q_i^2 + q_k^2))$, $(q_j q_k - q_r q_i)$
Row 3: $(q_i q_k - q_r q_j)$, $(q_j q_k + q_r q_i)$, $(\frac{1}{2} - (q_i^2 + q_k^2))$
We use the explicit matrix provided in the source notes [S1].

**Gradients for Scale $s$:**
The partial derivative of $M$ with respect to the scale components is diagonal. Since $M = RS$, $\frac{\partial M_{ij}}{\partial s_k}$ is non-zero only when $i=j=k$. Specifically:
$$ \frac{\partial M}{\partial s_X} = \begin{pmatrix} R_{11} & 0 & 0 \\ 0 & 0 & 0 \\ 0 & 0 & 0 \end{pmatrix}, \quad \frac{\partial M}{\partial s_Y} = \begin{pmatrix} 0 & 0 & 0 \\ 0 & R_{22} & 0 \\ 0 & 0 & 0 \end{pmatrix}, \quad \frac{\partial M}{\partial s_Z} = \begin{pmatrix} 0 & 0 & 0 \\ 0 & 0 & 0 \\ 0 & 0 & R_{33} \end{pmatrix} $$
The source does not explicitly write out the full matrix form for $\frac{\partial M}{\partial s_k}$ in the same detailed block as $q$, but implies it via $M=RS$. The gradient $\frac{\partial Z}{\partial s_k}$ is the dot product of $\frac{\partial Z}{\partial M}$ and $\frac{\partial M}{\partial s_k}$ [S1].

**Gradients for Quaternion $q$:**
The partial derivatives of $M$ with respect to the quaternion components are given by [S1]:

$$ \frac{\partial M}{\partial q_r} = 2 \begin{pmatrix} 0 & -s_Y q_k & s_Z q_j \\ s_X q_k & 0 & -s_Z q_i \\ -s_X q_j & s_Y q_i & 0 \end{pmatrix} $$

$$ \frac{\partial M}{\partial q_i} = 2 \begin{pmatrix} 0 & s_Y q_j & s_Z q_k \\ s_X q_k & -2s_Y q_i & -s_Z q_r \\ s_X q_k & s_Y q_r & -2s_Z q_j \end{pmatrix} $$

$$ \frac{\partial M}{\partial q_j} = 2 \begin{pmatrix} -2s_X q_k & s_Y q_j & s_Z q_r \\ s_X q_k & 0 & s_Z q_k \\ -s_X q_r & s_Y q_k & -2s_Z q_j \end{pmatrix} $$

$$ \frac{\partial M}{\partial q_k} = 2 \begin{pmatrix} -2s_X q_k & -s_Y q_j & s_Z q_i \\ s_X q_k & -2s_Y q_k & s_Z q_j \\ s_X q_i & s_Y q_j & 0 \end{pmatrix} $$

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def compute_gradients(dZ_dM, q, s):
    """
    Computes gradients of loss Z w.r.t. quaternion q and scale s.
    
    Args:
        dZ_dM: 3x3 matrix, gradient of loss w.r.t. M.
        q: 4-element vector [q_r, q_i, q_j, q_k].
        s: 3-element vector [s_X, s_Y, s_Z].
        
    Returns:
        dZ_dq: 4-element vector.
        dZ_ds: 3-element vector.
    """
    q_r, q_i, q_j, q_k = q
    s_X, s_Y, s_Z = s
    
    # Compute gradients for quaternion components
    # dZ_dq_k = sum(dZ_dM * dM/dq_k)
    
    # dM/dq_r
    dM_dqr = 2 * np.array([
        [0, -s_Y * q_k, s_Z * q_j],
        [s_X * q_k, 0, -s_Z * q_i],
        [-s_X * q_j, s_Y * q_i, 0]
    ])
    
    # dM/dq_i
    dM_dqi = 2 * np.array([
        [0, s_Y * q_j, s_Z * q_k],
        [s_X * q_k, -2 * s_Y * q_i, -s_Z * q_r],
        [s_X * q_k, s_Y * q_r, -2 * s_Z * q_j]
    ])
    
    # dM/dq_j
    dM_dqj = 2 * np.array([
        [-2 * s_X * q_k, s_Y * q_j, s_Z * q_r],
        [s_X * q_k, 0, s_Z * q_k],
        [-s_X * q_r, s_Y * q_k, -2 * s_Z * q_j]
    ])
    
    # dM/dq_k
    dM_dqk = 2 * np.array([
        [-2 * s_X * q_k, -s_Y * q_j, s_Z * q_i],
        [s_X * q_k, -2 * s_Y * q_k, s_Z * q_j],
        [s_X * q_i, s_Y * q_j, 0]
    ])
    
    # Gradient is the Frobenius inner product: sum(dZ_dM * dM/dq_component)
    dZ_dqr = np.sum(dZ_dM * dM_dqr)
    dZ_dqi = np.sum(dZ_dM * dM_dqi)
    dZ_dqj = np.sum(dZ_dM * dM_dqj)
    dZ_dqk = np.sum(dZ_dM * dM_dqk)
    
    dZ_dq = np.array([dZ_dqr, dZ_dqi, dZ_dqj, dZ_dqk])
    
    # Compute gradients for scale components
    # M = R * S, so dM/ds_X has R_11 at (0,0), etc.
    # dM/ds_X = [[R_00, 0, 0], [0, 0, 0], [0, 0, 0]]
    # dM/ds_Y = [[0, 0, 0], [0, R_11, 0], [0, 0, 0]]
    # dM/ds_Z = [[0, 0, 0], [0, 0, 0], [0, 0, R_22]]
    
    # We need R to compute these. R is computed from q.
    R = quaternion_to_rotation_matrix(q)
    
    dZ_dsX = dZ_dM[0, 0] * R[0, 0]
    dZ_dsY = dZ_dM[1, 1] * R[1, 1]
    dZ_dsZ = dZ_dM[2, 2] * R[2, 2]
    
    dZ_ds = np.array([dZ_dsX, dZ_dsY, dZ_dsZ])
    
    return dZ_dq, dZ_ds

def quaternion_to_rotation_matrix(q):
    """
    Converts quaternion to rotation matrix R as defined in [S1].
    """
    q_r, q_i, q_j, q_k = q
    R = 2 * np.array([
        [0.5 - (q_j**2 + q_k**2), q_i*q_j - q_r*q_k, q_i*q_k + q_r*q_j],
        [q_i*q_j + q_r*q_k, 0.5 - (q_i**2 + q_k**2), q_j*q_k - q_r*q_i],
        [q_i*q_k - q_r*q_j, q_j*q_k + q_r*q_i, 0.5 - (q_i**2 + q_k**2)]
    ])
    return R
```

**Edge Cases and Pitfalls:**
*   **Quaternion Normalization:** The derivation assumes $q$ is a unit quaternion. During optimization, $q$ may drift from unit length. The source mentions that "Deriving gradients for quaternion normalization is straightforward," implying a projection step or normalization constraint is applied, but does not provide the explicit gradient for the normalization operation itself [S1]. Implementers must ensure $q$ is normalized before computing $R(q)$ or apply a normalization gradient [G].
*   **Symmetry:** The covariance matrix $\Sigma$ is symmetric. The gradient $\frac{\partial Z}{\partial M}$ should be handled carefully to respect this symmetry if derived from $\Sigma$ [S1].
*   **Precision:** The formulas involve products of quaternion components and scales. Small scales or near-zero quaternion components may lead to precision issues.

## Operating characteristics
*   **Time Complexity:** $O(1)$ per Gaussian, as it involves fixed-size matrix multiplications and additions.
*   **Memory Complexity:** $O(1)$ per Gaussian, storing only the gradients for $q$ (4 floats) and $s$ (3 floats).
*   **Strengths:** Closed-form gradients allow for efficient backpropagation during training. No numerical differentiation is required.
*   **Limitations:** Requires the computation of the rotation matrix $R$ from $q$ to compute scale gradients. The quaternion normalization constraint is not explicitly handled in the gradient formulas provided [S1].

## Background
This method is part of the 3D Gaussian Splatting framework for real-time radiance field rendering. It was introduced by Kerbl, Kopanas, Leimkühler, and Drettos in 2023 [S1]. The derivation relies on the chain rule applied to the parameterization of the covariance matrix using quaternions and scaling vectors.

## Sources
[S1] 3D Gaussian Splatting for Real-Time Radiance Field Rendering, https://arxiv.org/abs/2308.04079, fetched 2026-10-26
