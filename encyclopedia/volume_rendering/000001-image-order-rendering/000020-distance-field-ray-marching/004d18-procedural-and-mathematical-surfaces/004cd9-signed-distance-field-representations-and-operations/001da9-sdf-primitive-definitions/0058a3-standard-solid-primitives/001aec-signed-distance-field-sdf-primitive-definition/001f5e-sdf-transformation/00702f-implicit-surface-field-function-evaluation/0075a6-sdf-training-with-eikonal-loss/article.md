# SDF Training with Eikonal Loss

Also called: Implicit Geometric Regularization, IGR, SDF Surface Training with Eikonal Loss, Neural SDF Training with Eikonal Regularization, Implicit neural representations with Eikonal regularization

## Overview
SDF Training with Eikonal Loss trains a neural network to represent a surface as the zero-level set of a Signed Distance Function (SDF) by enforcing the Eikonal equation ($||\nabla f|| = 1$) alongside data constraints [S8, unofficial][S9]. It solves the problem of reconstructing high-fidelity 3D shapes from raw point clouds or images without requiring pre-computed 3D supervision (like occupancy grids) [S8, unofficial]. Unlike occupancy-based methods, it explicitly regularizes the gradient magnitude to ensure the output is a valid metric distance field, distinguishing it from siblings like [DeepSDF Neural Network](3051-deepsdf-neural-network.md) which focuses on latent code optimization, and [SDF Optimization via Finite Differences](3086-sdf-optimization-via-finite-differences.md) which may lack the global geometric regularization [S8, S14, unofficial].

## How it works
**Inputs:** A set of oriented points $X = \{x_i\}$ with normals $N = \{n_i\}$, or posed images $I_j$ with projection matrices $W_j$.
**Outputs:** Network parameters $\theta$ defining the SDF $f(x; \theta)$.

**Data Structures:**
*   **MLP:** A Multi-Layer Perceptron $f: \mathbb{R}^3 \rightarrow \mathbb{R}$ with parameters $\theta$. Common architectures include 8 layers with 256 hidden units [S2] or configurations like D8-W45 (8 depth, 45 width) [S3, unofficial].
*   **Activation:** Softplus is recommended for smoothness [S8, unofficial].

**Steps:**
1.  **Preprocessing:** Center the input point cloud by subtracting its mean [S10, S6, unofficial]. Normalize the 3D space to a bounding box, typically $[-1, 1]^3$ [S1][S8, unofficial].
2.  **Sampling:**
    *   Sample points $X_{data}$ from the input point cloud [S10, unofficial].
    *   Sample points $X_{eik}$ uniformly from the bounding box (e.g., 1024 points) to enforce the Eikonal constraint everywhere [S1][S10, unofficial].
    *   *Optional:* For image-based training, sample pixels and cast rays [S2].
3.  **Forward Pass:**
    *   Query the network for distances and gradients at sampled points: $(d, \nabla d) = (f(x), \nabla_x f(x))$ for $x \in X_{data} \cup X_{eik}$ [S10, S6, unofficial].
    *   Compute gradients using automatic differentiation [S8, unofficial].
4.  **Loss Computation:**
    *   **Data Loss ($L_{data}$):** Enforces $f(x) \approx 0$ for points on the surface and aligns $\nabla f$ with normals $n$ [S9].
        $$L_{data} = \frac{1}{|X_{data}|} \sum_{x \in X_{data}} \left( f(x)^2 + (1 - \langle \nabla f(x), n(x) \rangle)^2 \right)$$
        [S9]
    *   **Eikonal Loss ($L_{eikonal}$):** Enforces unit gradient magnitude for all sampled points [S6, S10, unofficial].
        $$L_{eikonal} = \frac{1}{|X_{eik}|} \sum_{x \in X_{eik}} \left( \|\nabla f(x)\|_2 - 1 \right)^2$$
        [S10, unofficial]
        *Note: Some implementations use the absolute difference $|\|\nabla f(x)\|_2 - 1|$ [S1][S6, unofficial].*
    *   **Interior/Off-surface Regularization:** Penalizes points inside the object to ensure correct sign [S6, S10, unofficial].
        $$L_{inter} = \exp(-100 \cdot |f(x)|)$$
        Applied to $X_{eik}$ points [S6, S10, unofficial].
    *   **Total Loss:**
        $$L = L_{data} + \lambda_{eik} L_{eikonal} + \lambda_{inter} L_{inter}$$
        [S6, S10, unofficial]
5.  **Optimization:** Update $\theta$ using Adam optimizer [S10, S7, unofficial].
    *   Learning rate: $10^{-3}$ [S3, unofficial].
    *   Scheduler: LambdaLR or exponential decay [S7, S3, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import torch

def train_sdf_eikonal(model, optimizer, point_cloud, cfg):
    """
    Trains a neural SDF using Eikonal regularization.
    model: Neural network returning (distance, gradient)
    optimizer: PyTorch optimizer
    point_cloud: Tensor of shape (N, 3)
    cfg: Configuration object with training parameters
    """
    # 1. Center point cloud
    mean = torch.mean(point_cloud, dim=0, keepdim=True)
    points = point_cloud - mean
    
    # 2. Training loop
    for epoch in range(cfg.training.num_epochs):
        # 3. Sample data points
        data_indices = torch.randint(0, points.shape[0], (cfg.training.batch_size,))
        data_points = points[data_indices]
        
        # 4. Sample eikonal points (uniform in bounds)
        eikonal_points = torch.rand(cfg.training.batch_size, 3) * 2.0 - 1.0 # Assuming bounds [-1, 1]
        eikonal_points = eikonal_points + mean # Re-center if bounds are relative to mean
        
        # 5. Forward pass
        # Note: Requires grad for gradient computation
        data_points.requires_grad_(True)
        eikonal_points.requires_grad_(True)
        
        data_dist, data_grad = model.get_distance_and_gradient(data_points)
        eikonal_dist, eikonal_grad = model.get_distance_and_gradient(eikonal_points)
        
        # 6. Compute Losses
        # Data Loss: Distance to zero
        loss_data = torch.mean(data_dist ** 2)
        
        # Eikonal Loss: Gradient magnitude to 1
        grad_norms = torch.norm(eikonal_grad, dim=1, keepdim=True)
        loss_eikonal = torch.mean((grad_norms - 1.0) ** 2)
        
        # Interior Loss: Penalize points close to surface but not on it (optional but common)
        # Only apply to eikonal points to save compute, or specific subset
        interior_penalty = torch.exp(-1e2 * torch.abs(eikonal_dist))
        loss_interior = torch.mean(interior_penalty)
        
        # Total Loss
        loss = loss_data + \
               cfg.training.eikonal_weight * loss_eikonal + \
               cfg.training.inter_weight * loss_interior
               
        # 7. Backpropagation
        optimizer.zero_grad()
        loss.backward()
        optimizer.step()
        
        # 8. Learning Rate Scheduler Step (if applicable)
        if hasattr(cfg.training, 'lr_scheduler_step_size'):
            # Simplified step logic; actual implementation depends on scheduler type
            pass 
```

## Subtopics

### [MLP Distillation](4829-mlp-distillation.md)
Compresses a large SDF network into a smaller MLP by minimizing output discrepancy, transferring geometric knowledge from a teacher to a student network.

### [SDDF with Ellipsoidal Priors and Neural Residuals](10638-sddf-with-ellipsoidal-priors-and-neural-residuals.md)
Combines explicit ellipsoidal priors with neural residuals to handle occlusions and discontinuities, satisfying the directional Eikonal constraint by construction.

### [SDF-Instant-NGP Training](10640-sdf-instant-ngp-training.md)
Uses acceleration structures (hash grids) and includes a sign loss for free-space samples to enforce positive SDF values outside the surface.

### [TetraSDF Training Procedure](13235-tetrasdf-training-procedure.md)
Optimizes an SDF using a multi-resolution tetrahedral grid, combining L1 regression, Eikonal regularization, and weight normalization.

### [Regularization and Gradient Losses](17793-regularization-and-gradient-losses.md)
Extends the global Eikonal constraint to surface regression, spatial smoothness, and voxel-specific gradient consistency.

### [Geometric Loss Terms](17877-geometric-loss-terms.md)
Aggregates Eikonal, depth, normal, smoothness, and bubble losses to solve shape-radiance ambiguity and vanishing gradients.

### Bias Network for High-Fidelity Points
A network that captures geometric bias between volume rendering and point modeling, trained on high-fidelity points to enhance detail representation [S11, unofficial]. It filters points with low predicted variance ($\bar{\sigma}^2 < \epsilon$) to form a set $P_s$, then optimizes a bias MLP $f_b$ using L1 loss against ideal SDF values (0) on these points [S11, unofficial]. The final SDF is $f_{final} = f + f_b$ [S11, unofficial].

### SDF Training with Chamfer Distance
Trains a network to predict Unsigned Distance Function (UDF) values and gradients, projecting query points onto the implicit surface via gradient descent and minimizing Chamfer distance [S12, unofficial]. It requires `samples.requires_grad = True` before computing gradients to enable backpropagation [S12, unofficial].

### Implicit Surface to Volume Transformation
Represents geometry as a surface $S = \{x \mid F(x)=0\}$ and transforms it to volume by computing pseudo-density $\sigma_i = \Psi(F(x_i))$ using a cumulative distribution function (e.g., Laplace CDF with parameter $\beta$) [S13]. Unlike volumetric radiance fields, it restricts density prediction to spatial coordinates only [S13].

### L1 Regression Baseline for SDF
A baseline that trains an MLP to regress a pre-computed approximated SDF using L1 loss, serving as a fidelity comparison [S8, unofficial]. It approximates SDF locally using $s(x) = n^{*T}(x - y^*)$ and uses off-surface points, resulting in fewer details compared to the Eikonal method which uses only surface points and implicit regularization [S8, unofficial].

### SDF Regression
Regressing a feed-forward network to approximate the SDF induced by a surface, preferred over occupancy learning because SDFs are more useful for graphics operations like CSG [S14, unofficial].

## Operating characteristics
*   **Time Complexity:** Depends on MLP depth/width and number of sampled points. Inference time for D6-W60 MLP ranges from 15.0s to 25.3s per object on ShapeNet categories [S3, unofficial].
*   **Memory Complexity:** Proportional to the number of MLP parameters. Configurations like D8-W45 (360 neurons) or D20-W45 (900 neurons) are used [S3, unofficial].
*   **Strengths:** Works directly from raw point clouds without 3D supervision [S8, unofficial]. Implicit regularization favors smooth, plausible zero-level sets, avoiding bad critical solutions [S8, unofficial]. Produces higher fidelity and detail than L1 regression baselines [S8, unofficial].
*   **Limitations:** The zero-level isosurface only approximates the ground truth; quality depends on MLP capacity [S3, unofficial]. The solution to the Eikonal equation vanishing on point data is not unique; the method relies on implicit regularization to find a plausible SDF [S8, unofficial]. Thin objects are challenging due to rapid SDF flips and fewer samples [S2].
*   **When to use:** When reconstructing shapes from point clouds or images where a metric distance field is required, and pre-computed occupancy grids are unavailable.
*   **When not to use:** When exact topological correctness is critical without post-processing, or when real-time inference on very low-power hardware is required without acceleration structures.

## Background
Originated from the work on Implicit Geometric Regularization (IGR) by Gropp et al. (2020) [S8, unofficial]. The method was developed to compute high-fidelity implicit neural representations directly from raw data (point clouds) by encouraging the network to vanish on the input points and have unit norm gradients [S8, unofficial]. It builds on earlier work using implicit neural representations [S8, unofficial] and has been applied to datasets like ShapeNetCoreV1 (Rifle, Chair, Airplane, Sofa, Table) [S3, unofficial]. It differs from NeRF by using SDF values and Eikonal regularization instead of density and color prediction [S10, unofficial].

## Sources
[S1] ARAH: Animatable Volume Rendering of Articulated Human SDFs, https://doi.org/10.1007/978-3-031-19824-3_1, fetched 2026-10-26
[S2] Anti-Aliased Neural Implicit Surfaces with Encoding Level of Detail, https://doi.org/10.1145/3610548.3618197, fetched 2026-10-27
[S3] Analytic Marching: An Analytic Meshing Solution from Deep Implicit Surface Networks, https://arxiv.org/abs/2002.06597v1, fetched 2026-10-26 (unofficial)
[S4] DCCVT: Differentiable Clipped Centroidal Voronoi Tessellation, https://arxiv.org/abs/2601.13603v1, fetched 2026-10-26
[S5] Multi-scale hash encoding based neural geometry representation, https://doi.org/10.1007/s41095-023-0340-x, fetched 2026-10-26
[S6] omkarchittar/Neural_Surfaces, https://github.com/omkarchittar/Neural_Surfaces, fetched 2026-10-26 (unofficial)
[S7] Shreyas120/l3d_assignment3, https://github.com/Shreyas120/l3d_assignment3, fetched 2026-10-26 (unofficial)
[S8] Implicit Geometric Regularization for Learning Shapes, https://doi.org/10.48550/arxiv.2002.10099, fetched 2026-10-26 (unofficial)
[S9] Geometric implicit neural representations for signed distance functions, https://doi.org/10.1016/j.cag.2024.104085, fetched 2026-10-26
[S10] Fhz2346/Neural-Volume-Rendering-and-Surface-Rendering, https://github.com/Fhz2346/Neural-Volume-Rendering-and-Surface-Rendering, fetched 2026-10-27 (unofficial)
[S11] Point-NeuS: Point-Guided Neural Implicit Surface Reconstruction by Volume Rendering, https://doi.org/10.48550/arXiv.2310.07997, fetched 2026-10-30 (unofficial)
[S12] jjjkkyz/DCx, https://github.com/jjjkkyz/DCx, fetched 2026-10-26 (unofficial)
[S13] Radar Fields: An Extension of Radiance Fields to SAR, https://doi.org/10.1109/cvprw63382.2024.00061, fetched 2026-10-26
[S14] On the Effectiveness of Weight-Encoded Neural Implicit 3D Shapes, https://doi.org/10.48550/arxiv.2009.09808, fetched 2026-10-27 (unofficial)
