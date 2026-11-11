# SDF Training with Eikonal Loss

Also called: Eikonal Loss, Eikonal Regularization, Neural SDF Optimization, Neural SDF Training with Eikonal Regularization, Neural SDF Training with Point Cloud and Eikonal Regularization, SDF Regularization, SDF Surface Training with Eikonal Loss, VolSDF Training

## Overview
This method trains a Multi-Layer Perceptron (MLP) to predict a Signed Distance Function (SDF) from point cloud data by enforcing the Eikonal equation constraint, which requires the gradient magnitude of the predicted distance field to be unity everywhere [S2, unofficial][S3, unofficial]. It distinguishes itself from density-based neural rendering methods (like NeRF) by optimizing for geometric distance rather than volumetric opacity, ensuring the network learns a valid distance field rather than an arbitrary scalar field [S3, unofficial].

## How it works
**Inputs and Outputs:**
*   **Input:** A 3D point cloud dataset (vertices) and a bounding box defining the training volume.
*   **Output:** An optimized MLP that maps 3D coordinates to signed distance values, with gradients satisfying $\|\nabla \phi(x)\|_2 = 1$.

**Data Structures:**
*   **Point Cloud:** Tensor of shape $(N, 3)$ containing vertex coordinates.
*   **Bounding Box:** Defined by minimum and maximum coordinates (e.g., `cfg.training.bounds`).
*   **MLP:** A neural network (e.g., `NeuralSurface` class) taking 3D coordinates as input and outputting a scalar distance value [S2, unofficial].

**Numbered Steps:**

1.  **Data Preprocessing:** Load the point cloud vertices and center them by subtracting the mean coordinate:
    $$ \text{all\_points} = \text{all\_points} - \text{mean}(\text{all\_points}) $$ [S2, unofficial][S3, unofficial].
2.  **Sampling:**
    *   Sample $N$ points uniformly from the centered point cloud: $\mathcal{P}_{cloud} = \text{select\_random\_points}(\text{all\_points}, N)$ [S2, unofficial][S3, unofficial].
    *   Sample $M$ points uniformly within the training bounding box: $\mathcal{P}_{vol} = \text{get\_random\_points}(M, \text{bounds})$ [S2, unofficial][S3, unofficial].
3.  **Forward Pass (Point Cloud):** Query the MLP for distances at $\mathcal{P}_{cloud}$:
    $$ d_i = f(x_i), \quad x_i \in \mathcal{P}_{cloud} $$ [S2, unofficial][S3, unofficial].
4.  **Forward Pass (Volume):** Query the MLP for distances and gradients at $\mathcal{P}_{vol}$:
    $$ d_j = f(x_j), \quad g_j = \nabla f(x_j), \quad x_j \in \mathcal{P}_{vol} $$ [S2, unofficial][S3, unofficial].
    *   *Note:* Gradients are computed via automatic differentiation (e.g., `torch.autograd.grad`) [S3, unofficial].
5.  **Loss Computation:**
    *   **Point Loss:** Minimize the distance to the surface (ideally zero at points):
        $$ \mathcal{L}_{point} = \frac{1}{N} \sum_{i=1}^{N} |d_i| $$ [S3, unofficial].
    *   **Interior/Exterior Regularization:** Penalize points inside the object to enforce sign consistency (optional but common in VolSDF-style training):
        $$ \mathcal{L}_{inter} = \frac{1}{M} \sum_{j=1}^{M} \exp(-\lambda_{exp} |d_j|) $$ [S2, unofficial].
        *   $\lambda_{exp}$ is typically $100$ (written as `1e2`) [S2, unofficial][S3, unofficial].
    *   **Eikonal Loss:** Enforce unit gradient magnitude:
        $$ \mathcal{L}_{eikonal} = \frac{1}{M} \sum_{j=1}^{M} \left| \|g_j\|_2 - 1 \right| $$ [S2, unofficial]
        *   Alternative formulation using squared error:
            $$ \mathcal{L}_{eikonal} = \frac{1}{M} \sum_{j=1}^{M} (\|g_j\|_2 - 1)^2 $$ [S3, unofficial].
6.  **Total Loss:**
    $$ \mathcal{L}_{total} = \mathcal{L}_{point} + \lambda_{inter} \mathcal{L}_{inter} + \lambda_{eik} \mathcal{L}_{eikonal} $$ [S2, unofficial][S3, unofficial].
    *   $\lambda_{inter}$: Weight for interior regularization (`cfg.training.inter_weight`) [S2, unofficial][S3, unofficial].
    *   $\lambda_{eik}$: Weight for Eikonal regularization (`cfg.training.eikonal_weight`) [S2, unofficial][S3, unofficial].
7.  **Optimization:**
    *   Zero gradients: `optimizer.zero_grad()` [S2, unofficial][S3, unofficial].
    *   Backpropagate: `loss.backward()` [S2, unofficial][S3, unofficial].
    *   Step optimizer: `optimizer.step()` [S2, unofficial][S3, unofficial].
    *   Use Adam optimizer with a specified learning rate `lr` [S3, unofficial].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import torch
import torch.nn as nn

def eikonal_loss(gradients: torch.Tensor) -> torch.Tensor:
    """
    Computes the Eikonal loss enforcing unit gradient magnitude.
    gradients: Tensor of shape (N, 3) containing spatial gradients.
    """
    # Compute L2 norm along the last dimension (dim=1 for N,3)
    grad_norms = torch.norm(gradients, p=2, dim=1)
    # Return mean absolute difference from 1.0
    # Note: Some sources use squared difference ((norm - 1)**2).mean() [S3, unofficial]
    # This implementation uses absolute difference as per [S2, unofficial]
    return torch.mean(torch.abs(grad_norms - 1.0))

def train_sdf(model: nn.Module, optimizer: torch.optim.Optimizer, 
              point_cloud: torch.Tensor, bounds: tuple, 
              batch_size: int, num_epochs: int,
              lambda_inter: float = 1.0, lambda_eik: float = 1.0,
              lambda_exp: float = 100.0):
    """
    Trains an MLP to predict SDF with Eikonal regularization.
    """
    # 1. Center point cloud
    mean_pt = torch.mean(point_cloud, dim=0, keepdim=True)
    centered_points = point_cloud - mean_pt
    
    device = next(model.parameters()).device
    centered_points = centered_points.to(device)
    
    for epoch in range(num_epochs):
        # 2. Sample points from cloud
        indices = torch.randint(0, centered_points.shape[0], (batch_size,))
        cloud_points = centered_points[indices]
        
        # 3. Sample random points in volume
        # Uniform distribution within bounds
        vol_points = torch.rand(batch_size, 3, device=device)
        # Map [0,1] to bounds
        vol_points = vol_points * (bounds[1] - bounds[0]) + bounds[0]
        
        # 4. Forward pass: Cloud points
        cloud_points.requires_grad = True
        cloud_dists = model(cloud_points) # Shape (batch_size, 1)
        
        # 5. Forward pass: Volume points
        vol_points.requires_grad = True
        vol_dists, vol_grads = model.get_distance_and_gradient(vol_points)
        # get_distance_and_gradient returns (distances, gradients)
        # gradients shape: (batch_size, 3)
        
        # 6. Compute Losses
        
        # Point Loss: Distance to surface should be 0
        loss_point = torch.mean(torch.abs(cloud_dists))
        
        # Interior/Exterior Regularization (VolSDF style)
        # Penalizes points close to surface but not exactly on it, 
        # helping to define the sign and smoothness
        loss_inter = torch.mean(torch.exp(-lambda_exp * torch.abs(vol_dists)))
        
        # Eikonal Loss
        loss_eik = eikonal_loss(vol_grads)
        
        # Total Loss
        loss = loss_point + lambda_inter * loss_inter + lambda_eik * loss_eik
        
        # 7. Optimization Step
        optimizer.zero_grad()
        loss.backward()
        optimizer.step()

# Helper for model.get_distance_and_gradient (conceptual)
# In PyTorch:
# dist = model(x)
# grad = torch.autograd.grad(dist.sum(), x, create_graph=True)[0]
```

**Parameters:**
*   `lambda_eik` (or `cfg.training.eikonal_weight`): Weight for the Eikonal term. Typical values range from mild ($0.02$) to strong ($1.0$) [S1][S2, unofficial][S3, unofficial].
*   `lambda_inter` (or `cfg.training.inter_weight`): Weight for the interior regularization term [S2, unofficial][S3, unofficial].
*   `lambda_exp` (or `1e2`): Scaling factor in the exponential interior loss term [S2, unofficial][S3, unofficial].
*   `batch_size` (or `cfg.training.batch_size`): Number of points sampled per step [S2, unofficial][S3, unofficial].
*   `bounds` (or `cfg.training.bounds`): Spatial limits for sampling random points [S2, unofficial][S3, unofficial].
*   `lr`: Learning rate for the Adam optimizer [S3, unofficial].
*   `pretrain_iters`: Number of iterations for pretraining (e.g., with Sphere Loss) before Eikonal training [S3, unofficial].

**Edge Cases and Pitfalls:**
*   **Gradient Computation:** `requires_grad` must be set to `True` for the input points before computing gradients to enable backpropagation through the network [S5, unofficial].
*   **Precision:** Mild regularization ($\lambda_{Eik} = 0.02$) suppresses spurious outliers and topological noise but may cause a marginal decrease in surface precision (Chamfer Distance/F1) [S1].
*   **Initialization:** Pretraining with Sphere Loss can help initialize the implicit surface representation as a sphere before applying Eikonal regularization [S3, unofficial].

## Subtopics

### [Sphere Loss](2750-sphere-loss.md)
Sphere Loss is a regularization term used during the pretraining of Signed Distance Field (SDF) networks to initialize the implicit surface representation as a sphere. It minimizes the mean squared error between the network's predicted SDF values and the analytical Euclidean distance to a sphere surface of a specified radius.

### [MLP Distillation](4829-mlp-distillation.md)
MLP Distillation compresses a large Signed Distance Function (SDF) network into a smaller MLP by minimizing the output discrepancy between the two networks at sampled points. The teacher network $f$ uses 256 hidden channels, while the student network $f_{\text{small}}$ uses 16 hidden channels. The method transfers geometric knowledge from the teacher to the student over a fixed number of iterations before replacing the teacher for subsequent finetuning.

### [SDDF with Ellipsoidal Priors and Neural Residuals](10638-sddf-with-ellipsoidal-priors-and-neural-residuals.md)
This method learns a scene-level Signed Directional Distance Function (SDDF) by combining an explicit ellipsoidal prior with an implicit neural residual network. It solves the problem of reconstructing complex 3D environments from sensor data (LiDAR or RGB-D depth) while maintaining differentiability for novel view synthesis and avoiding the iterative sphere tracing required by standard SDFs. Unlike its siblings, it explicitly models occlusions via ellipsoids to handle discontinuities, while the neural network recovers fine details, satisfying the directional Eikonal constraint by construction.

### [SDF-Instant-NGP Training](10640-sdf-instant-ngp-training.md)
This method defines the training procedure for a Signed Distance Function (SDF) baseline network, specifically referred to as SDF-Instant-NGP in the source literature. It solves the problem of learning a continuous signed distance field from sparse or noisy geometric data by combining geometric supervision with physical regularization. It is distinguished from its parent method, [SDF Training with Eikonal Loss](2553-sdf-training-with-eikonal-loss.md), by the explicit inclusion of a sign loss term applied to free-space samples to enforce positive SDF values outside the surface, and by the specific hyperparameter configuration (batch size, learning rate, epochs) used for the SDF baseline in the RaDe-GS context.

### Bias Network for High-Fidelity Points
A network that captures geometric bias between volume rendering and point modeling, specifically trained on high-fidelity points to enhance detail representation [S4, unofficial]. It filters the point set $P$ to create $P_s$ containing only points with predicted variance below threshold $\varepsilon$: $P_s = \{x_P \mid x_P \in P, \bar{\sigma}^2(x_P) < \varepsilon\}$ [S4, unofficial]. The bias SDF value $f_b(x_P)$ is computed using a dedicated MLP: $f_b(x_P) = \text{MLP}_{bias}(x_P)$ [S4, unofficial]. The final SDF value is $f_{final}(x_P) = f(x_P) + f_b(x_P)$ [S4, unofficial]. The Bias network is optimized using L1 loss $\mathcal{L}_{bias} = \frac{1}{D_{P_s}} \sum_{x_P \in P_s} |f_\varepsilon(x_P) - f_{final}(x_P)|$, assuming ideal SDF values $f_\varepsilon(x_P)$ are 0 [S4, unofficial].

### SDF Training with Chamfer Distance
Trains a neural network to predict UDF values and gradients, projecting query points onto the implicit surface via gradient descent and minimizing Chamfer distance to ground truth points [S5, unofficial]. The projection is computed as $x_{\text{moved}} = x_{\text{sample}} - \frac{\nabla \text{UDF}(x_{\text{sample}})}{\|\nabla \text{UDF}(x_{\text{sample}})\|} \cdot \text{UDF}(x_{\text{sample}})$ [S5, unofficial]. A pitfall is that `samples.requires_grad = True` must be set before computing gradients to enable backpropagation through the network [S5, unofficial].

### Implicit Surface to Volume Transformation
Represents the scene geometry as a surface $S$ characterized by an implicit function $F$, where the surface is the zero-level set of $F$: $S = \{x \in \mathbb{R}^3 \mid F(x) = 0\}$ [S6]. It enforces the Eikonal property on the implicit function using a loss term that penalizes deviations of the gradient magnitude from 1 [S6]. The implicit surface is transformed into a volumetric model by computing a pseudo-density $\sigma_i$ from the SDF value using a cumulative distribution function: $\sigma_i = \Psi(F(x_i))$ [S6]. The transformation function $\Psi(d)$ is defined with the cumulative distribution function of the Laplace function:
$$
\Psi(d) = \begin{cases} 
\frac{1}{2} \exp(-d/\beta) & \text{for } d \geq 0 \\
1 - \frac{1}{2} \exp(d/\beta) & \text{for } d < 0 
\end{cases}
$$ [S6]. The parameter $\beta$ is an optimizable scale parameter [S6]. The set $\Omega$ contains points sampled near the surface as well as uniformly throughout the entire volume for Eikonal loss computation [S6]. Unlike volumetric radiance fields that predict density directly, surface models restrict the network to predict density based only on spatial coordinates $x$, while allowing color $c$ to depend on view-dependent features like direction $d$ [S6].

## Operating characteristics
*   **Time Complexity:** Depends on the number of training iterations, batch size, and MLP depth. Gradient computation adds overhead proportional to the number of hidden layers [S2, unofficial][S3, unofficial].
*   **Memory Complexity:** Proportional to the number of parameters in the MLP and the batch size of sampled points [S2, unofficial][S3, unofficial].
*   **Performance:** Mild Eikonal regularization ($\lambda_{Eik} = 0.02$) significantly improves Hausdorff Distance by suppressing spurious outliers, though it may cause a marginal decrease in Chamfer Distance and F1 score compared to unregularized baselines [S1].
*   **Strengths:** Ensures the learned field is a valid distance field (unit gradient), which is crucial for stable ray marching and surface extraction [S2, unofficial][S3, unofficial].
*   **Limitations:** Requires careful tuning of regularization weights; too strong regularization can smooth out fine details [S1].
*   **When to Use:** When training neural implicit surfaces from point clouds or when a valid distance field is required for downstream tasks like ray marching [S2, unofficial][S3, unofficial].
*   **When Not to Use:** When only density-based rendering is needed without explicit surface geometry, or when computational resources are extremely limited and gradient computation is prohibitive [S3, unofficial].

## Background
The method originates from the training of neural implicit surfaces, specifically popularized by works like VolSDF [S2, unofficial][S6]. It builds on the observation that signed distance functions satisfy the Eikonal equation $|\nabla \phi(x)| = 1$ [S2, unofficial][S3, unofficial]. The implementation details provided in the sources are largely derived from educational assignments and open-source repositories implementing neural surface rendering [S2, unofficial][S3, unofficial].

## Sources
[S1] DCCVT: Differentiable Clipped Centroidal Voronoi Tessellation, https://arxiv.org/abs/2601.13603v1, fetched 2026-10-26
[S2] omkarchittar/Neural_Surfaces, https://github.com/omkarchittar/Neural_Surfaces, fetched 2026-10-26 (unofficial)
[S3] Fhz2346/Neural-Volume-Rendering-and-Surface-Rendering, https://github.com/Fhz2346/Neural-Volume-Rendering-and-Surface-Rendering, fetched 2026-10-27 (unofficial)
[S4] Point-NeuS: Point-Guided Neural Implicit Surface Reconstruction by Volume Rendering, https://doi.org/10.48550/arXiv.2310.07997, fetched 2026-10-30 (unofficial)
[S5] jjjkkyz/DCx, https://github.com/jjjkkyz/DCx, fetched 2026-10-26 (unofficial)
[S6] Radar Fields: An Extension of Radiance Fields to SAR, https://doi.org/10.1109/cvprw63382.2024.00061, fetched 2026-10-26
