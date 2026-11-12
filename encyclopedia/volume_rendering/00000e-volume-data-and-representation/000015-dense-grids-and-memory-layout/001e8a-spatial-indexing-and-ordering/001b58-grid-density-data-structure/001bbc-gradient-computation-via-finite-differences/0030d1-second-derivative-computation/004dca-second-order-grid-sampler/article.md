# Second-order Grid Sampler

Also called: Double back-propagation for grid sampler, Hessian computation for SDF, Hessian computation for SDF via double backprop, Second-order derivative computation for SDF regularization, Second-order derivatives via double backpropagation

## Overview
The Second-order Grid Sampler is a specialized implementation of 3D trilinear interpolation designed to enable double back-propagation for computing the Hessian matrix of a Signed Distance Field (SDF) with respect to query coordinates and feature grid parameters [S1]. It solves the problem that standard deep learning frameworks (e.g., PyTorch) do not support second-order gradients through their native grid sampling operations [S1]. By explicitly deriving and implementing the analytical second-order derivatives, this method allows for the regularization of SDF gradients during neural network training, specifically computing blocks of the Hessian involving query positions and feature weights [S1]. It is distinct from finite-difference methods (see [Second Derivative Computation](6353-second-derivative-computation.md)) because it computes exact derivatives of the learned interpolation function rather than approximating derivatives of discrete grid values.

## How it works
The method computes the Hessian of the predicted SDF value $\phi$ with respect to the 3D query point $x$, the feature grid parameters $\theta$, and the geometry network parameters $\omega$ [S1]. The SDF is defined as $\phi(x, \theta, \omega) = f_\omega(z(\theta, x))$, where $z = V_\theta(x)$ is the tri-linearly interpolated feature vector at $x$ [S1]. To avoid higher-order tensors, the derivation considers derivatives for each individual feature dimension $z_i$ [S1].

### Inputs and Outputs
**Inputs:**
1.  Query point $x = [x, y, z]^T \in \mathbb{R}^3$, where coordinates are normalized to $[0, 1]$ within the voxel containing the point.
2.  Feature vector $\theta \in \mathbb{R}^8$ containing the feature values at the 8 vertices of the enclosing voxel, ordered as $[\theta_{000}, \theta_{001}, \theta_{010}, \theta_{011}, \theta_{100}, \theta_{101}, \theta_{110}, \theta_{111}]^T$ [S1].
3.  (Optional) Geometry network parameters $\omega$ and the function $f_\omega$ if the SDF output is processed by a network after interpolation.

**Outputs:**
1.  The Hessian block $\frac{\partial^2 \phi}{\partial x^2}$ (first row of the Hessian).
2.  The mixed Hessian block $\frac{\partial^2 \phi}{\partial x \partial \theta}$ (second row of the Hessian).
3.  The mixed Hessian block $\frac{\partial^2 \phi}{\partial x \partial \omega}$ (third row of the Hessian) [S1].

### Data Structures
*   **Coefficient Vector $w(x)$**: An 8-dimensional vector representing the trilinear interpolation weights for the 8 voxel corners [S1].
*   **Jacobian Matrix $\frac{\partial w}{\partial x}$**: An $8 \times 3$ matrix containing the first-order derivatives of the weights with respect to $x, y, z$ [S1].

### Formulas
The trilinearly interpolated feature $f$ at point $x$ is:
$$ f = \theta^T w(x) $$
where $w(x) \in \mathbb{R}^8$ is:
$$ w(x) = \begin{bmatrix} (1-x)(1-y)(1-z) \\ (1-x)(1-y)z \\ (1-x)y(1-z) \\ (1-x)yz \\ x(1-y)(1-z) \\ x(1-y)z \\ xy(1-z) \\ xyz \end{bmatrix} $$

The Jacobian of $f$ with respect to $x$ is:
$$ \frac{\partial f}{\partial x} = \theta^T \frac{\partial w(x)}{\partial x} $$
The Jacobian matrix $\frac{\partial w}{\partial x}$ is an $8 \times 3$ matrix with columns $J_1, J_2, J_3$:
$$ J_1 = \frac{\partial w}{\partial x} = \begin{bmatrix} -(1-y)(1-z) \\ -(1-y)z \\ -y(1-z) \\ -yz \\ (1-y)(1-z) \\ (1-y)z \\ y(1-z) \\ yz \end{bmatrix}, \quad J_2 = \frac{\partial w}{\partial y} = \begin{bmatrix} -(1-x)(1-z) \\ -(1-x)z \\ (1-x)(1-z) \\ (1-x)z \\ -x(1-z) \\ -xz \\ x(1-z) \\ xz \end{bmatrix}, \quad J_3 = \frac{\partial w}{\partial z} = \begin{bmatrix} -(1-x)(1-y) \\ (1-x)(1-y) \\ -(1-x)y \\ (1-x)y \\ -x(1-y) \\ x(1-y) \\ -xy \\ xy \end{bmatrix} $$

The mixed second derivative block $\frac{\partial^2 f}{\partial x \partial \theta}$ is the transpose of the Jacobian of $w$ with respect to $x$:
$$ \frac{\partial^2 f}{\partial x \partial \theta} = \left( \frac{\partial w}{\partial x} \right)^T $$

The diagonal elements of the Hessian $\frac{\partial^2 f}{\partial x^2}$ are zero because each column $J_i$ only contains variables from the other two dimensions [S1]. The off-diagonal elements are:
$$ h_{12} = h_{21} = (1-z)(\theta_{000} + \theta_{110} - \theta_{010} - \theta_{100}) + z(\theta_{001} + \theta_{111} - \theta_{011} - \theta_{101}) $$
$$ h_{13} = h_{31} = (1-y)(\theta_{000} + \theta_{101} - \theta_{001} - \theta_{100}) + y(\theta_{010} + \theta_{111} - \theta_{011} - \theta_{110}) $$
$$ h_{23} = h_{32} = (1-x)(\theta_{000} + \theta_{011} - \theta_{001} - \theta_{010}) + x(\theta_{100} + \theta_{111} - \theta_{101} - \theta_{110}) $$

The full second derivative of the SDF $\phi$ w.r.t $x$ (Eq. 3 in [S1]) is:
$$ \frac{\partial^2 \phi}{\partial x^2} = \sum_{i=1}^C \left( \frac{\partial^2 f_\omega(z)}{\partial z_i^2} \left(\frac{\partial z_i}{\partial x}\right)^2 + \frac{\partial f_\omega(z)}{\partial z_i} \frac{\partial^2 z_i}{\partial x^2} \right) $$
The mixed second derivative w.r.t $x$ and $\theta$ (Eq. 4 in [S1]) is:
$$ \frac{\partial^2 \phi}{\partial x \partial \theta} = \sum_{i=1}^C \frac{\partial^2 f_\omega(z)}{\partial z_i^2} \frac{\partial z_i}{\partial \theta} \frac{\partial z_i}{\partial x} $$
The mixed second derivative w.r.t $x$ and $\omega$ (Eq. 5 in [S1]) is:
$$ \frac{\partial^2 \phi}{\partial x \partial \omega} = \sum_{i=1}^C \left( \frac{\partial^2 f_\omega(z)}{\partial z_i^2} \frac{\partial z_i}{\partial \omega} \frac{\partial z_i}{\partial x} + \frac{\partial f_\omega(z)}{\partial z_i} \frac{\partial^2 z_i}{\partial x \partial \omega} \right) $$

### Pseudocode
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def second_order_grid_sampler(x, theta, f_omega_derivatives=None):
    """
    Computes the Hessian blocks for the SDF regularizer.
    
    Args:
        x: Tensor of shape (..., 3), query coordinates [x, y, z] in [0, 1].
        theta: Tensor of shape (..., 8), feature values at voxel corners.
        f_omega_derivatives: Optional tuple (df_dz, d2f_dz2) if f_omega is not identity.
                             df_dz: shape (..., C)
                             d2f_dz2: shape (..., C)
                             If None, assumes f_omega is identity (df_dz=1, d2f_dz2=0).
    
    Returns:
        hessian_xx: Tensor of shape (..., 3, 3), second derivative w.r.t. x.
        hessian_xtheta: Tensor of shape (..., 3, 8), mixed derivative w.r.t. x and theta.
        hessian_xomega: Tensor of shape (..., 3, ...) if f_omega depends on omega, else None.
    """
    x_val, y_val, z_val = x[..., 0], x[..., 1], x[..., 2]
    
    # 1. Compute Jacobian of weights w.r.t x (8x3)
    # J1 = dw/dx, J2 = dw/dy, J3 = dw/dz
    J1 = torch.stack([
        -(1-y_val)*(1-z_val), -(1-y_val)*z_val, -y_val*(1-z_val), -y_val*z_val,
        (1-y_val)*(1-z_val), (1-y_val)*z_val, y_val*(1-z_val), y_val*z_val
    ], dim=-1)
    
    J2 = torch.stack([
        -(1-x_val)*(1-z_val), -(1-x_val)*z_val, (1-x_val)*(1-z_val), (1-x_val)*z_val,
        -x_val*(1-z_val), -x_val*z_val, x_val*(1-z_val), x_val*z_val
    ], dim=-1)
    
    J3 = torch.stack([
        -(1-x_val)*(1-y_val), (1-x_val)*(1-y_val), -(1-x_val)*y_val, (1-x_val)*y_val,
        -x_val*(1-y_val), x_val*(1-y_val), -x_val*y_val, x_val*y_val
    ], dim=-1)
    
    # Stack to get (..., 8, 3)
    J_w_x = torch.stack([J1, J2, J3], dim=-1)
    
    # 2. Compute hessian_xtheta = (dw/dx)^T = J_w_x^T -> (..., 3, 8)
    hessian_xtheta = J_w_x.transpose(-1, -2)
    
    # 3. Compute hessian_xx
    # Diagonal elements are 0.
    # Off-diagonals:
    # h12 = (1-z)(t000 + t110 - t010 - t100) + z(t001 + t111 - t011 - t101)
    # Indices: 0:000, 1:001, 2:010, 3:011, 4:100, 5:101, 6:110, 7:111
    
    term1_12 = (1-z_val) * (theta[..., 0] + theta[..., 6] - theta[..., 2] - theta[..., 4])
    term2_12 = z_val * (theta[..., 1] + theta[..., 7] - theta[..., 3] - theta[..., 5])
    h12 = term1_12 + term2_12
    
    term1_13 = (1-y_val) * (theta[..., 0] + theta[..., 5] - theta[..., 1] - theta[..., 4])
    term2_13 = y_val * (theta[..., 2] + theta[..., 7] - theta[..., 3] - theta[..., 6])
    h13 = term1_13 + term2_13
    
    term1_23 = (1-x_val) * (theta[..., 0] + theta[..., 3] - theta[..., 1] - theta[..., 2])
    term2_23 = x_val * (theta[..., 4] + theta[..., 7] - theta[..., 5] - theta[..., 6])
    h23 = term1_23 + term2_23
    
    # Construct 3x3 matrix
    # h11=h22=h33=0
    hessian_xx = torch.zeros(*x.shape[:-1], 3, 3, device=x.device)
    hessian_xx[..., 0, 1] = h12
    hessian_xx[..., 1, 0] = h12
    hessian_xx[..., 0, 2] = h13
    hessian_xx[..., 2, 0] = h13
    hessian_xx[..., 1, 2] = h23
    hessian_xx[..., 2, 1] = h23
    
    # 4. Apply chain rule for f_omega if present
    # If f_omega is identity, hessian_xx and hessian_xtheta are final.
    # If f_omega exists, we need df/dz and d2f/dz2.
    if f_omega_derivatives is not None:
        df_dz, d2f_dz2 = f_omega_derivatives
        # df_dz shape: (..., C), d2f_dz2 shape: (..., C)
        # z is the interpolated feature. In the simplest case (identity), z = theta^T w.
        # But generally, z_i is the i-th component of the interpolated vector.
        # The formula sums over i=1 to C.
        # For identity f_omega: df/dz = 1, d2f/dz2 = 0.
        # Then d2phi/dx2 = sum_i ( 0 * (dz_i/dx)^2 + 1 * d2z_i/dx2 ) = sum_i d2z_i/dx2
        # d2z_i/dx2 corresponds to the i-th element of the Hessian_xx derived above for scalar theta.
        # Since theta is vector, we sum over the feature dimension.
        
        # For hessian_xtheta: sum_i ( d2f/dz2_i * (dz_i/dtheta) * (dz_i/dx) )
        # dz_i/dtheta is the i-th row of J_w_x^T? No, z_i = theta_i^T w.
        # dz_i/dtheta is a vector with 1 at index i? No, theta is (..., 8).
        # z = theta * w (dot product over 8).
        # dz/dtheta = w (shape 8).
        # So d2phi/dx dtheta = sum_i d2f/dz2_i * w * (dz_i/dx)
        
        # This part requires knowing the structure of f_omega. 
        # The source [S1] states Eq 5 is not needed if PyTorch supports double backprop through MLP.
        # Eq 3 and 4 need manual implementation.
        # If f_omega is identity, the derived hessian_xx and hessian_xtheta are the answer.
        pass
        
    return hessian_xx, hessian_xtheta
```

### Parameters
*   **Feature Dimension**: The derivation assumes 1-D features for simplicity but generalizes to any feature dimension $C$ as the derivation is equivalent for all dimensions [S1].
*   **Coordinate System**: Query coordinates $x, y, z$ are assumed to be normalized to $[0, 1]$ within the local voxel [S1].

### Edge Cases and Pitfalls
*   **PyTorch Limitations**: Standard PyTorch `grid_sampler` does not support second-order gradients (double back-propagation) through the sampling operation [S1]. This method must be implemented manually as shown [S1].
*   **Higher-Order Tensors**: The method avoids higher-order tensors by summing over individual feature dimensions $z_i$ [S1].
*   **Diagonal Zeros**: The diagonal elements of the Hessian $\frac{\partial^2 f}{\partial x^2}$ are strictly zero due to the linearity of the interpolation with respect to each coordinate individually [S1].

## Subtopics

### [Mean Curvature Grid Computation](3126-mean-curvature-grid-computation.md)
Computes mean curvature from the Hessian trace. Choose when surface smoothness is required.

### [Divergence Grid Computation](3151-divergence-grid-computation.md)
Computes divergence of a vector field. Choose for fluid dynamics or vector field regularization.

### [Laplacian Grid Computation](3160-laplacian-grid-computation.md)
Computes the Laplacian (trace of Hessian). Choose for diffusion or smoothing.

### [Depth and Image Gradient Regularization](6439-depth-and-image-gradient-computation.md)
Regularizes first-order gradients in 2D/3D. Choose for image/depth consistency.

### [Eikonal Omega Term Computation](13240-eikonal-omega-term-computation.md)
Computes terms related to the Eikonal equation. Choose for SDF normalization.

## Operating characteristics
*   **Time Complexity**: $O(1)$ per query point, as it involves a fixed number of arithmetic operations (8 multiplications and additions for weights, constant time for Hessian elements).
*   **Memory Complexity**: $O(1)$ auxiliary memory per query point.
*   **Strengths**: Provides exact analytical second-order derivatives for trilinear interpolation, enabling gradient-based regularization of SDFs in neural networks. Avoids numerical instability of finite differences.
*   **Limitations**: Only applicable to trilinear interpolation on a grid. Does not support higher-order interpolation schemes without re-derivation. Requires manual implementation in frameworks lacking native double-backprop support for grid sampling.
*   **When to use**: When training neural networks that predict SDFs from feature grids and gradient regularization (e.g., Eikonal loss) is required.
*   **When not to use**: When using finite-difference methods on discrete grids (use [Second Derivative Computation](6353-second-derivative-computation.md)) or when higher-order interpolation is needed.

## Background
The Second-order Grid Sampler was introduced by Jingwen Wang, Tymoteusz Bleja, and Lourdes Agapito in the paper "GO-Surf: Neural Feature Grid Optimization for Fast, High-Fidelity RGB-D Surface Reconstruction" [S1]. It was developed to address the limitation of PyTorch's grid sampler not supporting double back-propagation, which is necessary for computing the Hessian of the SDF with respect to query coordinates and feature parameters for regularization purposes [S1]. The method is part of the GO-Surf pipeline, which uses neural feature grids for fast RGB-D surface reconstruction [S1].

## Sources
[S1] GO-Surf: Neural Feature Grid Optimization for Fast, High-Fidelity RGB-D Surface Reconstruction, https://doi.org/10.1109/3dv57658.2022.00055, fetched 2026-10-29
