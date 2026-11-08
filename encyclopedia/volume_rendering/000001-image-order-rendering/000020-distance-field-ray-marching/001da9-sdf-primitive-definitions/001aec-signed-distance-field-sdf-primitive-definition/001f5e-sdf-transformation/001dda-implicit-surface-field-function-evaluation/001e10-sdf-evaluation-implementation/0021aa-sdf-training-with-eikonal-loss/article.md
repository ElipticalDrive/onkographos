# SDF Training with Eikonal Loss

Also called: Eikonal Loss, Eikonal Regularization, Neural SDF Optimization, Neural SDF Training with Eikonal Regularization, Neural SDF Training with Point Cloud and Eikonal Regularization, SDF Regularization, SDF Surface Training with Eikonal Loss, VolSDF Training

## Overview
SDF Training with Eikonal Loss trains a Multi-Layer Perceptron (MLP) to predict a Signed Distance Function (SDF) from point cloud data by enforcing the Eikonal equation, which requires the gradient magnitude of the SDF to be unity everywhere [S2, unofficial][S3, unofficial]. Unlike methods that predict density directly, this approach ensures the network output represents a valid geometric distance field, suppressing spurious outliers and topological noise while maintaining surface precision [S1]. It sets itself apart from [Batched SDF Inference](2329-batched-sdf-inference.md) by focusing on the optimization of the field's geometric properties rather than efficient evaluation, and from [DeepSDF Neural Network](3051-deepsdf-neural-network.md) by explicitly regularizing the gradient norm rather than relying solely on latent code priors.

## How it works
**Inputs and Outputs:**
*   **Input:** A 3D point cloud $P$ (vertices), a bounding box $\Omega$ for sampling, and hyperparameters including batch size $N$, learning rate, and regularization weights.
*   **Output:** An optimized MLP $\phi(x)$ that predicts the signed distance to the surface, where $\phi(x) = 0$ on the surface.

**Data Structures:**
*   **MLP $\phi(x)$:** A feedforward neural network taking a 3D coordinate vector $x \in \mathbb{R}^3$ and outputting a scalar distance $d \in \mathbb{R}$.
*   **Point Cloud Tensor:** Shape $(M, 3)$, containing $M$ vertices.
*   **Gradient Tensor:** Shape $(N, 3)$, containing the spatial gradients $\nabla \phi(x)$ for $N$ sampled points.

**Numbered Steps:**

1.  **Data Preparation:** Load the point cloud vertices and center them by subtracting the mean coordinate:
    $$ x_{centered} = x_{raw} - \frac{1}{M}\sum_{j=1}^{M} x_{raw}^{(j)} $$
    [S2, unofficial][S3, unofficial]

2.  **Model Initialization:** Initialize the MLP $\phi$ and an optimizer (e.g., Adam) with a specified learning rate [S3, unofficial].

3.  **Pretraining (Optional):** If applicable, pretrain the network using [Sphere Loss](2750-sphere-loss.md) to initialize the implicit surface as a sphere of known radius, minimizing the MSE between predicted distances and the analytical Euclidean distance to the sphere [S1][S2, unofficial].

4.  **Sampling:**
    *   Sample $N$ points $x_i$ uniformly from the input point cloud $P$.
    *   Sample $N$ points $x_j$ uniformly within the training bounding box $\Omega$ [S2, unofficial][S3, unofficial].

5.  **Forward Pass:**
    *   Query the MLP for distances and gradients at the sampled points.
    *   For point cloud points $x_i$: compute $d_i = \phi(x_i)$ and $\nabla d_i = \nabla \phi(x_i)$.
    *   For bounding box points $x_j$: compute $d_j = \phi(x_j)$ and $\nabla d_j = \nabla \phi(x_j)$ [S2, unofficial][S3, unofficial].

6.  **Loss Computation:**
    *   **Point Cloud Loss:** Compute the mean squared error of the distances for points on the cloud (target is 0):
        $$ \mathcal{L}_{point} = \frac{1}{N} \sum_{i=1}^{N} d_i^2 $$
        [S3, unofficial]
    *   **Eikonal Loss:** Compute the deviation of the gradient magnitude from 1. Sources provide two variants:
        *   L1 variant:
            $$ \mathcal{L}_{eikonal} = \frac{1}{N} \sum_{k=1}^{N} \left| \| \nabla \phi(x_k) \|_2 - 1 \right| $$
            [S2, unofficial]
        *   L2 variant:
            $$ L_{eikonal} = \frac{1}{N} \sum_{k=1}^{N} (\| \nabla \phi(x_k) \|_2 - 1)^2 $$
            [S3, unofficial]
    *   **Interior/Exterior Regularization (VolSDF style):** Penalize points inside the object to prevent the SDF from being negative deep inside or positive deep outside, using an exponential term:
        $$ \mathcal{L}_{inter} = \frac{1}{N} \sum_{j=1}^{N} \exp(-100 \cdot |d_j|) $$
        [S2, unofficial]

7.  **Total Loss:** Combine the terms with weights $\lambda_{point}$, $\lambda_{eikonal}$, and $\lambda_{inter}$:
    $$ \mathcal{L}_{total} = \lambda_{point} \mathcal{L}_{point} + \lambda_{eikonal} \mathcal{L}_{eikonal} + \lambda_{inter} \mathcal{L}_{inter} $$
    [S2, unofficial][S3, unofficial]

8.  **Optimization:** Perform backpropagation to update the MLP weights:
    $$ \theta \leftarrow \theta - \eta \nabla_\theta \mathcal{L}_{total} $$
    [S2, unofficial][S3, unofficial]

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import torch
import torch.nn as nn

def eikonal_loss(gradients: torch.Tensor) -> torch.Tensor:
    """
    Computes the Eikonal loss enforcing unit gradient magnitude.
    Uses L2 norm squared difference from 1.0 as per [S3, unofficial].
    """
    # Compute L2 norm of gradients along the last dimension (features)
    grad_norms = torch.norm(gradients, p=2, dim=-1)
    # Return mean squared difference from 1.0
    return torch.mean((grad_norms - 1.0) ** 2)

def train_sdf_with_eikonal(model: nn.Module, point_cloud: torch.Tensor, 
                           bounds: tuple, optimizer: torch.optim.Optimizer, 
                           batch_size: int, eikonal_weight: float, 
                           inter_weight: float, num_epochs: int):
    """
    Trains an SDF MLP using Eikonal regularization.
    """
    # Center point cloud
    mean = torch.mean(point_cloud, dim=0, keepdim=True)
    centered_points = point_cloud - mean
    
    device = next(model.parameters()).device
    
    for epoch in range(num_epochs):
        # 1. Sample points from point cloud
        indices = torch.randint(0, centered_points.shape[0], (batch_size,))
        points_pc = centered_points[indices].to(device)
        
        # 2. Sample random points in bounds
        # Assuming bounds is (min_val, max_val)
        min_val, max_val = bounds
        points_rand = torch.rand(batch_size, 3, device=device) * (max_val - min_val) + min_val
        
        # 3. Forward pass for point cloud points
        points_pc.requires_grad = True
        dist_pc = model(points_pc) # Shape: (batch_size, 1)
        # Compute gradients for point cloud points
        grad_pc = torch.autograd.grad(
            dist_pc.sum(), points_pc, create_graph=True, retain_graph=True
        )[0] # Shape: (batch_size, 3)
        
        # 4. Forward pass for random points
        points_rand.requires_grad = True
        dist_rand, grad_rand = model.get_distance_and_gradient(points_rand) 
        # Note: get_distance_and_gradient must return both d and grad. 
        # If not available, compute grad manually as above.
        # For this self-contained example, we assume a helper or manual computation:
        if not hasattr(model, 'get_distance_and_gradient'):
            dist_rand = model(points_rand)
            grad_rand = torch.autograd.grad(
                dist_rand.sum(), points_rand, create_graph=True, retain_graph=True
            )[0]
            
        # 5. Compute Losses
        # Point loss: distance should be 0
        loss_point = torch.mean(dist_pc ** 2)
        
        # Eikonal loss: gradient norm should be 1
        loss_eikonal = eikonal_loss(grad_rand)
        
        # Interior/Exterior regularization (VolSDF style)
        # Penalize points far from zero level set
        loss_inter = torch.mean(torch.exp(-1e2 * torch.abs(dist_rand)))
        
        # Total loss
        loss = loss_point + eikonal_weight * loss_eikonal + inter_weight * loss_inter
        
        # 6. Backpropagate
        optimizer.zero_grad()
        loss.backward()
        optimizer.step()
```

**Parameters:**
*   `eikonal_weight`: Weight for the gradient magnitude regularization term. Typical values are tuned per dataset [S2, unofficial][S3, unofficial].
*   `inter_weight`: Weight for the exponential interior loss term. Typical values are tuned per dataset [S2, unofficial][S3, unofficial].
*   `batch_size`: Number of points sampled per optimization step.
*   `bounds`: Spatial limits (min, max) for sampling random eikonal points.
*   `lr`: Learning rate for the Adam optimizer.
*   $\lambda_{Eik}$: In some contexts, a single regularization weight is used. Mild regularization ($\lambda_{Eik} = 0.02$) suppresses spurious outliers with marginal precision loss [S1].

**Edge Cases and Pitfalls:**
*   **Gradient Computation:** `requires_grad` must be set to `True` for the input points before computing gradients to enable backpropagation through the network [S5, unofficial].
*   **Precision:** High regularization weights (e.g., $\lambda_{Eik} = 2.0$) can cause severe degradation in surface precision (Chamfer Distance/F1) and topological consistency [S1].
*   **Boundary Conditions:** Sampling must cover both the surface (point cloud) and the volume (bounding box) to enforce the Eikonal constraint everywhere, not just on the surface [S2, unofficial][S3, unofficial].

## Subtopics

### [Sphere Loss](2750-sphere-loss.md)
Sphere Loss is a regularization term used during the pretraining of Signed Distance Field (SDF) networks to initialize the implicit surface representation as a sphere. It minimizes the mean squared error between the network's predicted SDF values and the analytical Euclidean distance to a sphere surface of a specified radius. Choose this when initializing an SDF network to provide a stable starting point for subsequent Eikonal training.

### [MLP Distillation](4829-mlp-distillation.md)
MLP Distillation compresses a large, high-capacity Signed Distance Function (SDF) network into a smaller, more efficient MLP while preserving geometric accuracy. It solves the problem of high inference latency and memory usage associated with large neural implicit representations by transferring knowledge from a "teacher" model to a "student" model. Unlike standard training which learns geometry from scratch, this method minimizes the output discrepancy between the two networks at sampled points, allowing the small network to inherit the teacher's geometric priors. It sets itself apart from sibling methods like Sphere Loss by focusing on direct output-space alignment rather than geometric constraints on the field's curvature or local shape.

### Bias Network for High-Fidelity Points
A network that captures geometric bias between volume rendering and point modeling, specifically trained on high-fidelity points to enhance detail representation [S4, unofficial]. It filters the point set $P$ to create $P_s$ containing only points with predicted variance below threshold $\epsilon$ ($P_s = \{x_P | x_P \in P, \bar{\sigma}^2(x_P) < \epsilon\}$). The bias SDF value $f_b(x_P)$ is computed via a dedicated MLP, and the final SDF is $f_{final}(x_P) = f(x_P) + f_b(x_P)$. The Bias network is optimized using L1 loss $\mathcal{L}_{bias} = \frac{1}{D_{P_s}} \sum_{x_P \in P_s} |f_\epsilon(x_P) - f_{final}(x_P)|$, assuming ideal SDF values $f_\epsilon(x_P)$ are 0 [S4, unofficial]. Choose this when high-frequency details are lost in standard SDF training due to volume rendering ambiguities.

### SDF Training with Chamfer Distance
Trains a neural network to predict Unsigned Distance Function (UDF) values and gradients, projecting query points onto the implicit surface via gradient descent and minimizing Chamfer distance to ground truth points [S5, unofficial]. It moves a sample point $x_{sample}$ to the surface using $x_{moved} = x_{sample} - \frac{\nabla \text{UDF}(x_{sample})}{\|\nabla \text{UDF}(x_{sample})\|} \cdot \text{UDF}(x_{sample})$. A pitfall is that `samples.requires_grad = True` must be set before computing gradients [S5, unofficial]. Choose this when working with unsigned distances or when the signed nature of the SDF is not strictly required or known.

### Implicit Surface to Volume Transformation
Represents the scene geometry as a surface $S$ characterized by an implicit function $F$, where $S = \{x \in \mathbb{R}^3 \mid F(x) = 0\}$ [S6]. It enforces the Eikonal property and transforms the implicit surface into a volumetric model by computing a pseudo-density $\sigma_i = \Psi(F(x_i))$ using a Laplace cumulative distribution function:
$$ \Psi(d) = \begin{cases} \frac{1}{2} \exp(-d/\beta) & \text{for } d \geq 0 \\ 1 - \frac{1}{2} \exp(d/\beta) & \text{for } d < 0 \end{cases} $$
where $\beta$ is an optimizable scale parameter [S6]. Choose this when integrating SDF training into a volume rendering pipeline (like VolSDF) that requires density values for ray marching.

## Operating characteristics
*   **Time Complexity:** $O(N \cdot L)$ per iteration, where $N$ is the number of sampled points and $L$ is the depth of the MLP. Gradient computation adds a backward pass of similar cost.
*   **Memory Complexity:** $O(N \cdot D)$, where $D$ is the MLP parameter count, plus $O(N)$ for storing gradients and intermediate tensors during backpropagation.
*   **Measured Performance:** Mild Eikonal regularization ($\lambda_{Eik} = 0.02$) significantly improves Hausdorff Distance by suppressing spurious outliers, with only a marginal decrease in Chamfer Distance [S1].
*   **Strengths:** Enforces geometric validity (unit gradient), improves topological consistency, and stabilizes training for implicit surfaces.
*   **Limitations:** Requires careful tuning of regularization weights; high weights degrade precision. Does not inherently handle color or texture without extension (e.g., [SDF and Material Optimization Pipeline](4458-sdf-and-material-optimization-pipeline.md)).
*   **When to Use:** When training neural implicit surfaces from point clouds or when a valid distance field is required for downstream tasks like ray marching or collision detection.
*   **When Not to Use:** When inference latency is critical and a smaller network is needed (use [MLP Distillation](4829-mlp-distillation.md)), or when the input is images rather than point clouds (use [NeRF](link) or [VolSDF](link) variants).

## Background
The method originates from the training of neural implicit surfaces, specifically popularized by works such as VolSDF [S6] and various academic assignments and repositories [S2, unofficial][S3, unofficial]. It leverages the mathematical property of Signed Distance Functions that their gradient magnitude is always 1 (the Eikonal equation $|\nabla \phi(x)| = 1$). By adding this as a regularization term during MLP optimization, the network is guided to learn a true distance field rather than an arbitrary scalar field that happens to be zero at the surface. This approach differs from NeRF by using SDF values and Eikonal regularization instead of density and color prediction [S3, unofficial].

## Sources
[S1] DCCVT: Differentiable Clipped Centroidal Voronoi Tessellation, https://arxiv.org/abs/2601.13603v1, fetched 2026-10-26
[S2] omkarchittar/Neural_Surfaces, https://github.com/omkarchittar/Neural_Surfaces, fetched 2026-10-26 (unofficial)
[S3] Fhz2346/Neural-Volume-Rendering-and-Surface-Rendering, https://github.com/Fhz2346/Neural-Volume-Rendering-and-Surface-Rendering, fetched 2026-10-27 (unofficial)
[S4] Point-NeuS: Point-Guided Neural Implicit Surface Reconstruction by Volume Rendering, https://doi.org/10.48550/arXiv.2310.07997, fetched 2026-10-30 (unofficial)
[S5] jjjkkyz/DCx, https://github.com/jjjkkyz/DCx, fetched 2026-10-26 (unofficial)
[S6] Radar Fields: An Extension of Radiance Fields to SAR, https://doi.org/10.1109/cvprw63382.2024.00061, fetched 2026-10-26
