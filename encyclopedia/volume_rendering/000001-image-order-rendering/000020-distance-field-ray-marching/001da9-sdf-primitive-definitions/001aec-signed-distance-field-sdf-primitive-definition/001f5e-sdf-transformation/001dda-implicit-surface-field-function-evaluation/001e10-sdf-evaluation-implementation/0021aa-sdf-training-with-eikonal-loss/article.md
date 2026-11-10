# SDF Training with Eikonal Loss

Also called: Eikonal Loss, Eikonal Regularization, Neural SDF Optimization, Neural SDF Training with Eikonal Regularization, Neural SDF Training with Point Cloud and Eikonal Regularization, SDF Regularization, SDF Surface Training with Eikonal Loss, VolSDF Training

## Overview
SDF Training with Eikonal Loss trains a Multi-Layer Perceptron (MLP) to predict a Signed Distance Function (SDF) from point cloud data by enforcing the Eikonal equation constraint, which requires the gradient magnitude of the predicted field to be unity everywhere [S2, unofficial][S3, unofficial]. Unlike methods that predict density directly (e.g., NeRF) or rely solely on geometric fitting, this approach ensures the learned implicit function represents a valid distance field, improving topological consistency and suppressing spurious outliers [S1][S2, unofficial]. It builds on [SDF Evaluation Implementation](1658-sdf-evaluation-implementation.md) by adding gradient-based regularization during the optimization phase.

## How it works
**Inputs and Outputs:**
*   **Input:** A 3D point cloud $P$ (vertices), a bounding box $\Omega$ for sampling, and an MLP architecture (typically defined in a `NeuralSurface` class).
*   **Output:** An optimized MLP $\phi(x)$ that approximates the SDF, where $\phi(x) \approx 0$ on the surface, $\phi(x) > 0$ outside, and $\phi(x) < 0$ inside.

**Data Structures:**
*   **Point Cloud:** Tensor of shape $(N, 3)$ containing vertex coordinates.
*   **Bounding Box:** Defined by minimum and maximum coordinates (e.g., `cfg.training.bounds`).
*   **MLP:** A feedforward network taking 3D coordinates and outputting a scalar distance value.

**Numbered Steps:**

1.  **Data Preparation:**
    *   Load the point cloud vertices.
    *   Center the point cloud by subtracting the mean:
        $$ P_{centered} = P - \text{mean}(P) $$
    *   [G] Ensure points are on the correct device (e.g., GPU) and require gradients if backpropagation through the network is needed for gradient computation.

2.  **Sampling:**
    *   Sample $N_p$ points uniformly from the centered point cloud $P_{centered}$. Let these be $X_p$.
    *   Sample $N_e$ points uniformly within the training bounding box $\Omega$. Let these be $X_e$.

3.  **Forward Pass (Point Cloud Loss):**
    *   Query the MLP for distances at $X_p$: $D_p = \phi(X_p)$.
    *   Compute the point cloud loss (L2 norm of distances, assuming target is 0):
        $$ \mathcal{L}_{point} = \frac{1}{N_p} \sum_{i=1}^{N_p} \| D_{p,i} \|_2 $$
    *   [G] In many implementations, this is simply the mean of the absolute values or squared values of the distances if the network outputs signed distances and we want to pull them to zero. The source code uses `.norm(dim=-1).mean()`.

4.  **Forward Pass (Eikonal Loss):**
    *   Query the MLP for distances and gradients at $X_e$: $(D_e, G_e) = (\phi(X_e), \nabla \phi(X_e))$.
    *   Compute the gradient norm for each point: $g_i = \| G_{e,i} \|_2$.
    *   Compute the Eikonal loss. Sources provide two variants:
        *   L1 variant [S2, unofficial]:
            $$ \mathcal{L}_{eikonal} = \frac{1}{N_e} \sum_{i=1}^{N_e} | g_i - 1 | $$
        *   L2 variant [S3, unofficial]:
            $$ L_{eikonal} = \frac{1}{N_e} \sum_{i=1}^{N_e} ( g_i - 1 )^2 $$
    *   [G] The L2 variant is more common in deep learning frameworks for smoother gradients near the target.

5.  **Interior/Free-Space Regularization (Optional but common in VolSDF-style training):**
    *   Penalize points that are far from the surface or inside the object to enforce the sign constraint and smoothness.
    *   Compute an exponential interior loss:
        $$ \mathcal{L}_{inter} = \text{mean}(\exp(-100 \cdot |D_e|)) $$
    *   This term is small when $|D_e|$ is large (far from surface) and large when close to the surface, effectively regularizing the gradient near the zero-crossing.

6.  **Total Loss and Optimization:**
    *   Combine losses with weights:
        $$ \mathcal{L}_{total} = \mathcal{L}_{point} + \lambda_{eik} \mathcal{L}_{eikonal} + \lambda_{inter} \mathcal{L}_{inter} $$
    *   Perform backpropagation: `optimizer.zero_grad()`, `loss.backward()`, `optimizer.step()`.

**Formulas:**
*   **Eikonal Equation:** $|\nabla \phi(x)| = 1$ [S2, unofficial][S3, unofficial].
*   **Eikonal Loss (L2):** $L_{eikonal} = \frac{1}{N} \sum_{i=1}^{N} (\|\nabla \phi(x_i)\|_2 - 1)^2$ [S3, unofficial].
*   **Eikonal Loss (L1):** $\mathcal{L}_{eikonal} = \frac{1}{N} \sum_{i=1}^{N} |\|\nabla \phi(x_i)\|_2 - 1|$ [S2, unofficial].

**Parameters:**
*   $\lambda_{eik}$ (or `cfg.training.eikonal_weight`): Weight for the Eikonal loss. Typical values range from 0.01 to 1.0. [S1] uses $\lambda_{Eik} = 0.02$ for mild regularization.
*   $\lambda_{inter}$ (or `cfg.training.inter_weight`): Weight for the interior regularization. [S3, unofficial] uses a scaling factor of $100$ inside the exponential.
*   `batch_size`: Number of points sampled per step.
*   `bounds`: Spatial limits for uniform sampling.
*   `lr`: Learning rate for the optimizer (e.g., Adam).

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import torch
import torch.nn as nn
import torch.optim as optim

def compute_eikonal_loss_l2(gradients: torch.Tensor) -> torch.Tensor:
    """
    Computes the L2 Eikonal loss: mean((||grad|| - 1)^2).
    Args:
        gradients: Tensor of shape (N, 3) containing spatial gradients.
    Returns:
        Scalar loss value.
    """
    grad_norms = torch.norm(gradients, dim=1, keepdim=True) # Shape (N, 1)
    loss = (grad_norms - 1.0) ** 2
    return loss.mean()

def compute_eikonal_loss_l1(gradients: torch.Tensor) -> torch.Tensor:
    """
    Computes the L1 Eikonal loss: mean(||grad|| - 1).
    Args:
        gradients: Tensor of shape (N, 3) containing spatial gradients.
    Returns:
        Scalar loss value.
    """
    grad_norms = torch.norm(gradients, dim=1, keepdim=True)
    loss = torch.abs(grad_norms - 1.0)
    return loss.mean()

def train_sdf_with_eikonal_loss(model, point_cloud, bounds, config):
    """
    Trains an SDF MLP using Eikonal regularization.
    
    Args:
        model: NeuralSurface MLP instance.
        point_cloud: Tensor of shape (N, 3) containing vertex coordinates.
        bounds: Tuple (min_coord, max_coord) for uniform sampling.
        config: Object containing hyperparameters (batch_size, weights, etc.).
    """
    # 1. Center point cloud
    mean = torch.mean(point_cloud, dim=0, keepdim=True)
    centered_points = point_cloud - mean
    
    # 2. Initialize optimizer
    optimizer = optim.Adam(model.parameters(), lr=config.lr)
    
    # 3. Training Loop
    for epoch in range(config.num_epochs):
        optimizer.zero_grad()
        
        # Sample points from point cloud
        # Select random indices
        idx_p = torch.randint(0, centered_points.shape[0], (config.batch_size,))
        points_p = centered_points[idx_p]
        
        # Sample random points in bounds
        # Uniform distribution in [min, max]
        min_coord, max_coord = bounds
        points_e = torch.rand(config.batch_size, 3, device=point_cloud.device) * (max_coord - min_coord) + min_coord
        
        # 4. Forward Pass: Point Cloud Loss
        # Get distances for point cloud samples
        dists_p = model.get_distance(points_p) # Shape (N, 1) or (N,)
        # Loss is mean L2 norm of distances (target is 0)
        loss_point = torch.norm(dists_p, dim=-1).mean()
        
        # 5. Forward Pass: Eikonal Loss
        # Get distances and gradients for random points
        dists_e, grads_e = model.get_distance_and_gradient(points_e)
        
        # Compute Eikonal Loss (using L2 variant as per [S3, unofficial])
        loss_eikonal = compute_eikonal_loss_l2(grads_e)
        
        # 6. Interior Regularization (VolSDF style)
        # Penalize points far from surface to enforce sign/structure
        # exp(-100 * |dist|)
        loss_inter = torch.exp(-100.0 * torch.abs(dists_e)).mean()
        
        # 7. Combine Losses
        total_loss = (loss_point + 
                      config.eikonal_weight * loss_eikonal + 
                      config.inter_weight * loss_inter)
        
        # 8. Backpropagate
        total_loss.backward()
        optimizer.step()
        
        # Optional: Log loss
        if epoch % 100 == 0:
            print(f"Epoch {epoch}, Loss: {total_loss.item():.4f}")
```

**Edge Cases and Pitfalls:**
*   **Gradient Computation:** Ensure `requires_grad=True` is set for the sampled points if computing gradients via `torch.autograd.grad` manually, though most modern frameworks handle this if the model parameters require gradients. [S5, unofficial] explicitly notes `samples.requires_grad = True`.
*   **Precision:** The Eikonal constraint is sensitive to the scale of the input coordinates. Centering and normalizing the point cloud is critical [S2, unofficial][S3, unofficial].
*   **Degenerate Input:** If the point cloud has no volume (e.g., a flat sheet), the SDF may not be well-defined inside. The interior regularization helps but may not fully resolve topological ambiguities without additional constraints [S1].

## Subtopics

### [Sphere Loss](2750-sphere-loss.md)
Sphere Loss is a regularization term used during the pretraining of Signed Distance Field (SDF) networks to initialize the implicit surface representation as a sphere. It minimizes the mean squared error between the network's predicted SDF values and the analytical Euclidean distance to a sphere surface of a specified radius.

### [MLP Distillation](4829-mlp-distillation.md)
MLP Distillation compresses a large, high-capacity Signed Distance Function (SDF) network into a smaller, more efficient MLP while preserving geometric accuracy. It solves the problem of high inference latency and memory usage associated with large neural implicit representations by transferring knowledge from a "teacher" model to a "student" model. Unlike standard training which learns geometry from scratch, this method minimizes the output discrepancy between the two networks at sampled points, allowing the small network to inherit the teacher's geometric priors. It sets itself apart from sibling methods like Sphere Loss by focusing on direct output-space alignment rather than geometric constraints on the field's curvature or local shape.

### [SDDF with Ellipsoidal Priors and Neural Residuals](10638-sddf-with-ellipsoidal-priors-and-neural-residuals.md)
This method learns a scene-level Signed Directional Distance Function (SDDF) using a hybrid explicit-implicit representation to enable fast, single-query differentiable novel view synthesis and geometric reconstruction. It solves the challenge of learning directional distance fields for complex scenes by combining an explicit ellipsoidal prior, which approximates coarse object shapes and occlusions, with an implicit neural residual network that refines fine details. Unlike standard SDFs that require iterative sphere tracing, or NeRFs that optimize volume density for photometry, this approach directly predicts the signed distance to the nearest surface along a specific viewing direction in a single forward pass.

### [SDF-Instant-NGP Training](10640-sdf-instant-ngp-training.md)
This method defines the training procedure for a Signed Distance Function (SDF) baseline network, specifically referred to as SDF-Instant-NGP in the source literature. It solves the problem of learning a continuous signed distance field from sparse or noisy geometric data by combining geometric supervision with physical regularization. It is distinguished from its parent method, [SDF Training with Eikonal Loss](2553-sdf-training-with-eikonal-loss.md), by the explicit inclusion of a sign loss term applied to free-space samples to enforce positive SDF values outside the surface, and by the specific hyperparameter configuration (batch size, learning rate, epochs) used for the SDF baseline in the RaDe-GS context.

### Bias Network for High-Fidelity Points
A network that captures geometric bias between volume rendering and point modeling, specifically trained on high-fidelity points to enhance detail representation [S4, unofficial]. It filters the point set $P$ to create $P_s$ containing only points with predicted variance below threshold $\epsilon$: $P_s = \{x_P | x_P \in P, \bar{\sigma}^2(x_P) < \epsilon\}$ [S4, unofficial]. It computes the bias SDF value $f_b(x_P)$ for points in $P_s$ using a dedicated MLP: $f_b(x_P) = MLP_{bias}(x_P)$ [S4, unofficial]. The final SDF value is $f_{final}(x_P) = f(x_P) + f_b(x_P)$ [S4, unofficial]. It optimizes the Bias network using L1 loss $\mathcal{L}_{bias} = \frac{1}{D_{P_s}} \sum_{x_P \in P_s} |f_\epsilon(x_P) - f_{final}(x_P)|$, assuming ideal SDF values $f_\epsilon(x_P)$ are 0 [S4, unofficial].

### SDF Training with Chamfer Distance
Trains a neural network to predict UDF values and gradients, projecting query points onto the implicit surface via gradient descent and minimizing Chamfer distance to ground truth points [S5, unofficial]. It moves sample points $x_{sample}$ to the surface using $x_{moved} = x_{sample} - \frac{\nabla \text{UDF}(x_{sample})}{\|\nabla \text{UDF}(x_{sample})\|} \cdot \text{UDF}(x_{sample})$ [S5, unofficial]. A pitfall is that `samples.requires_grad = True` must be set before computing gradients to enable backpropagation through the network [S5, unofficial].

### Implicit Surface to Volume Transformation
Represents the scene geometry as a surface $S$ characterized by an implicit function $F$, where the surface is the zero-level set of $F$: $S = \{x \in \mathbb{R}^3 \mid F(x) = 0\}$ [S6]. It enforces the Eikonal property on the implicit function using a loss term that penalizes deviations of the gradient magnitude from 1 [S6]. It transforms the implicit surface into a volumetric model by computing a pseudo-density $\sigma_i$ from the SDF value using a cumulative distribution function: $\sigma_i = \Psi(F(x_i))$ [S6]. The transformation function $\Psi(d)$ is defined as:
$$ \Psi(d) = \begin{cases} \frac{1}{2} \exp(-d/\beta) & \text{for } d \geq 0 \\ 1 - \frac{1}{2} \exp(d/\beta) & \text{for } d < 0 \end{cases} $$
where $\beta$ is an optimizable scale parameter [S6]. Points are sampled near the surface and uniformly throughout the volume for Eikonal loss computation [S6]. Unlike volumetric radiance fields that predict density directly, surface models restrict the network to predict density based only on spatial coordinates $x$, while allowing color $c$ to depend on view-dependent features like direction $d$ [S6].

## Operating characteristics
*   **Time Complexity:** Training time depends on the number of epochs, batch size, and MLP depth. Gradient computation adds overhead compared to simple forward passes.
*   **Memory Complexity:** Scales with the number of parameters in the MLP and the batch size of sampled points.
*   **Performance:** Mild Eikonal regularization ($\lambda_{Eik} = 0.02$) significantly improves Hausdorff Distance (HD) by suppressing spurious outliers, with a marginal decrease in Chamfer Distance (CD) and F1 score [S1].
*   **Strengths:** Enforces geometric consistency (unit gradient), leading to cleaner topologies and better surface reconstruction from point clouds.
*   **Limitations:** Requires careful tuning of regularization weights. Can smooth out fine details if the weight is too high.
*   **When to Use:** When training neural SDFs from point clouds or when topological consistency is more important than absolute local precision.
*   **When Not to Use:** When real-time inference latency is critical and the MLP is too large, or when the input data is extremely noisy and requires different regularization strategies.

## Background
The method originates from the training of neural implicit surfaces, specifically popularized by works like VolSDF [S6] and implementations in educational repositories like `Neural_Surfaces` [S2, unofficial] and `Neural-Volume-Rendering-and-Surface-Rendering` [S3, unofficial]. The Eikonal equation $|\nabla \phi(x)| = 1$ is a fundamental property of signed distance functions, and enforcing it during training helps the network learn a valid distance field rather than an arbitrary scalar field. This approach differs from NeRF by using SDF values and Eikonal regularization instead of density and color prediction [S3, unofficial].

## Sources
[S1] DCCVT: Differentiable Clipped Centroidal Voronoi Tessellation, https://arxiv.org/abs/2601.13603v1, fetched 2026-10-26
[S2] omkarchittar/Neural_Surfaces, https://github.com/omkarchittar/Neural_Surfaces, fetched 2026-10-26 (unofficial)
[S3] Fhz2346/Neural-Volume-Rendering-and-Surface-Rendering, https://github.com/Fhz2346/Neural-Volume-Rendering-and-Surface-Rendering, fetched 2026-10-27 (unofficial)
[S4] Point-NeuS: Point-Guided Neural Implicit Surface Reconstruction by Volume Rendering, https://doi.org/10.48550/arXiv.2310.07997, fetched 2026-10-30 (unofficial)
[S5] jjjkkyz/DCx, https://github.com/jjjkkyz/DCx, fetched 2026-10-26 (unofficial)
[S6] Radar Fields: An Extension of Radiance Fields to SAR, https://doi.org/10.1109/cvprw63382.2024.00061, fetched 2026-10-26
