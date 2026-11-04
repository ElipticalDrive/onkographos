# SDF Training with Eikonal Loss

Also called: Eikonal Loss, Neural SDF Optimization, Neural SDF Training with Eikonal Regularization, Neural SDF Training with Point Cloud and Eikonal Regularization, SDF Regularization, SDF Surface Training with Eikonal Loss, VolSDF Training

## Overview
SDF Training with Eikonal Loss optimizes a neural implicit Signed Distance Field (SDF) to match 3D point cloud data by enforcing the Eikonal equation constraint [S1, unofficial]. It trains a Multi-Layer Perceptron (MLP) to predict the signed distance from any 3D point to a target surface defined by a point cloud [S1, unofficial]. The method combines a data fidelity term that matches the point cloud with an Eikonal regularization term that enforces geometric consistency [S1, S2, unofficial].

## How it works
### Inputs and Outputs
*   **Input**: A 3D point cloud $P$ representing the target surface [S1, unofficial].
*   **Output**: An MLP model $\phi(x)$ that outputs the signed distance value for any input coordinate $x \in \mathbb{R}^3$ [S1, unofficial].

### Data Structures
*   **MLP Model**: A standard feedforward neural network defined within a `NeuralSurface` class [S1, S2, unofficial]. The network takes a 3D coordinate as input and outputs a scalar distance value [S1, unofficial].
*   **Point Cloud Data**: A tensor of shape $(N, 3)$ containing vertex coordinates [S2, unofficial].
*   **Training Configuration**:
    *   `cfg.training.batch_size`: Number of points sampled per optimization step [S1, unofficial].
    *   `cfg.training.bounds`: Spatial limits defining the bounding box for sampling random points [S1, unofficial].
    *   `cfg.training.eikonal_weight`: Weight for the gradient Eikonal loss term [S2, unofficial].
    *   `cfg.training.inter_weight`: Weight for the interior regularization loss term [S2, unofficial].
    *   `lr`: Learning rate for the optimizer [S2, unofficial].

### Numbered Steps
1.  **Data Preprocessing**: Load the point cloud vertices and center them by subtracting the mean coordinate [S1, unofficial].
    $$ P_{centered} = P - \text{mean}(P) $$
2.  **Model Initialization**: Initialize the MLP and an optimizer (typically Adam) with the specified learning rate [S2, unofficial].
3.  **Pretraining (Optional)**: Before training on the specific point cloud, the network may be pretrained using [Sphere Loss](2750-sphere-loss.md) [S1, S2, unofficial].
4.  **Sampling**:
    *   Sample a batch of points $X_{pc}$ uniformly from the centered point cloud $P_{centered}$ [S2, unofficial].
    *   Sample a batch of points $X_{rand}$ uniformly within the training bounding box `cfg.training.bounds` [S1, unofficial].
5.  **Forward Pass (Point Cloud)**:
    *   Query the MLP for distances at $X_{pc}$: $D_{pc} = \phi(X_{pc})$ [S2, unofficial].
    *   Compute the point cloud loss as the mean L2 norm of the distances (expecting zero at the surface):
        $$ \mathcal{L}_{pc} = \frac{1}{N_{pc}} \sum_{i=1}^{N_{pc}} \| D_{pc,i} \|_2 $$
6.  **Forward Pass (Random Points)**:
    *   Query the MLP for distances and gradients at $X_{rand}$ [S1, unofficial].
    *   The gradients $\nabla \phi(X_{rand})$ are computed using automatic differentiation [G].
    *   Let $D_{rand} = \phi(X_{rand})$ and $G_{rand} = \nabla \phi(X_{rand})$ [S1, unofficial].
7.  **Loss Computation**:
    *   **Interior Regularization**: Penalize points inside the object (negative distance) to encourage the zero-level set to be the surface [S1, unofficial]. This uses an exponential weighting based on the distance magnitude:
        $$ \mathcal{L}_{inter} = \text{mean}(\exp(-100 \cdot |D_{rand}|)) $$
        The factor $100$ (written as `1e2` in code) scales the exponential decay [S2, unofficial].
    *   **Eikonal Loss**: Enforce that the gradient magnitude is 1 [S1, S2, unofficial]. Sources provide two variants of this loss:
        *   Variant 1 (L1): $\mathcal{L}_{eikonal} = \frac{1}{N} \sum_{i=1}^{N} |\|\nabla \phi(x_i)\|_2 - 1|$ [S1, unofficial].
        *   Variant 2 (L2): $\mathcal{L}_{eikonal} = \frac{1}{N} \sum_{i=1}^{N} (\|\nabla \phi(x_i)\|_2 - 1)^2$ [S2, unofficial].
    *   **Total Loss**:
        $$ \mathcal{L}_{total} = \mathcal{L}_{pc} + \mathcal{L}_{inter} \cdot w_{inter} + \mathcal{L}_{eikonal} \cdot w_{eikonal} $$
        where $w_{inter}$ is `cfg.training.inter_weight` and $w_{eikonal}$ is `cfg.training.eikonal_weight` [S2, unofficial].
8.  **Optimization**: Zero gradients, backpropagate the total loss, and step the optimizer [S1, S2, unofficial].

### Pseudocode
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import torch
import torch.nn as nn

def get_random_points(batch_size: int, bounds: list, device: str = 'cpu') -> torch.Tensor:
    # Uniform sampling within bounds [min_x, max_x, min_y, max_y, min_z, max_z]
    min_vals = torch.tensor(bounds[0::2], dtype=torch.float32, device=device)
    max_vals = torch.tensor(bounds[1::2], dtype=torch.float32, device=device)
    return torch.rand(batch_size, 3, device=device) * (max_vals - min_vals) + min_vals

def compute_eikonal_loss(gradients: torch.Tensor) -> torch.Tensor:
    # L2 variant from [S2, unofficial]; L1 variant is also used [S1, unofficial]
    grad_norms = torch.norm(gradients, p=2, dim=-1)
    return torch.mean((grad_norms - 1.0) ** 2)

def train_sdf_eikonal(model: nn.Module, point_cloud: torch.Tensor, config: dict):
    # 1. Center point cloud
    mean_pos = torch.mean(point_cloud, dim=0, keepdim=True)
    centered_points = point_cloud - mean_pos
    
    # 2. Initialize optimizer
    optimizer = torch.optim.Adam(model.parameters(), lr=config['lr'])
    
    # 3. Training Loop
    for epoch in range(config['num_epochs']):
        optimizer.zero_grad()
        
        # 4. Sample points from point cloud
        pc_indices = torch.randint(0, centered_points.shape[0], (config['batch_size'],))
        points_pc = centered_points[pc_indices]
        
        # 5. Query distances for point cloud points
        distances_pc = model.get_distance(points_pc)
        
        # 6. Compute Point Cloud Loss (L2 norm of distances)
        loss_pc = torch.norm(distances_pc, dim=-1).mean()
        
        # 7. Sample random points in bounds
        rand_points = get_random_points(config['batch_size'], config['bounds'], point_cloud.device)
        
        # 8. Query distances and gradients for random points
        distances_rand = model.get_distance(rand_points)
        gradients_rand = torch.autograd.grad(distances_rand.sum(), rand_points, create_graph=True)[0]
        
        # 9. Compute Interior Regularization Loss
        inter_loss = torch.exp(-1e2 * torch.abs(distances_rand)).mean()
        
        # 10. Compute Eikonal Loss
        eikonal_loss_val = compute_eikonal_loss(gradients_rand)
        
        # 11. Combine Losses
        total_loss = (loss_pc + 
                      inter_loss * config['inter_weight'] + 
                      eikonal_loss_val * config['eikonal_weight'])
        
        # 12. Backpropagate and Optimize
        total_loss.backward()
        optimizer.step()
```

### Parameters and Typical Values
*   **Learning Rate (`lr`)**: Configurable learning rate for the Adam optimizer [S2, unofficial].
*   **Batch Size**: Number of points sampled per optimization step [S1, unofficial].
*   **Eikonal Weight (`eikonal_weight`)**: Scales the gradient magnitude regularization term [S2, unofficial].
*   **Interior Weight (`inter_weight`)**: Scales the exponential interior loss term [S2, unofficial].
*   **Exponential Scale**: The factor $100$ (`1e2`) in the interior loss term is fixed in the reference implementation [S2, unofficial].

### Edge Cases and Pitfalls
*   **Precision**: The exponential term $\exp(-100 \cdot |d|)$ can underflow for points far from the surface, resulting in zero gradients [S1, unofficial].
*   **Boundary Conditions**: Random points sampled outside the object's convex hull may not contribute meaningfully to the Eikonal constraint if the interior loss dominates [S1, unofficial]. The bounding box must be large enough to cover the object but not excessively large to waste computation [S1, unofficial].
*   **Gradient Vanishing**: If the network fails to learn a proper SDF, gradients may become zero or very small, causing the Eikonal loss to saturate [S1, unofficial].

## Subtopics

### [Sphere Loss](2750-sphere-loss.md)
Sphere Loss is a regularization term used during the pretraining of Signed Distance Field (SDF) networks to initialize the implicit surface representation as a sphere. It minimizes the mean squared error between the network's predicted SDF values and the analytical Euclidean distance to a sphere surface of a specified radius. Unlike general Eikonal loss which enforces gradient constraints on arbitrary shapes, Sphere Loss provides a strong geometric prior by forcing the network to learn a specific, simple topology, stabilizing subsequent training on complex shapes.

### Bias Network for High-Fidelity Points
A network that captures geometric bias between volume rendering and point modeling, specifically trained on high-fidelity points to enhance detail representation [S3, unofficial]. It filters the point set $P$ to create $P_s$ containing only points with predicted variance below threshold $\epsilon$: $P_s = \{x_P \mid x_P \in P, \bar{\sigma}^2(x_P) < \epsilon\}$ [S3, unofficial]. The bias SDF value $f_b(x_P)$ is computed using a dedicated MLP: $f_b(x_P) = \text{MLP}_{bias}(x_P)$ [S3, unofficial]. The final SDF value is the sum of the base SDF $f(x_P)$ and the bias SDF: $f_{final}(x_P) = f(x_P) + f_b(x_P)$ [S3, unofficial]. The Bias network is optimized using L1 loss $\mathcal{L}_{bias} = \frac{1}{D_{P_s}} \sum_{x_P \in P_s} |f_\varepsilon(x_P) - f_{final}(x_P)|$, assuming ideal SDF values $f_\varepsilon(x_P)$ are 0 [S3, unofficial]. This allows the model to learn high-frequency details from reliable points while remaining independent of volume rendering gradients [S3, unofficial].

## Operating characteristics
*   **Time Complexity**: Linear with respect to the number of sampled points $N$ and the depth/width of the MLP [S1, S2, unofficial]. Each training step requires $N$ forward passes and $N$ backward passes through the network [S1, S2, unofficial].
*   **Memory Complexity**: Linear with respect to $N$ for storing gradients and intermediate activations [S1, S2, unofficial]. The model size is constant [S1, S2, unofficial].
*   **Performance**: The Eikonal constraint significantly improves the geometric accuracy of the reconstructed surface compared to methods without it, particularly in regions with sparse point cloud data [S1, S2, unofficial]. However, it requires careful tuning of loss weights to prevent the regularization from overpowering the data fidelity term [S1, S2, unofficial].
*   **Strengths**: Produces valid signed distance fields with smooth gradients [S1, S2, unofficial]. Enables efficient ray marching for rendering [S1, S2, unofficial].
*   **Limitations**: Sensitive to hyperparameter tuning (loss weights, learning rate) [S1, S2, unofficial]. May struggle with highly complex geometries if the MLP capacity is insufficient [S1, S2, unofficial]. Pretraining step adds overhead [S1, S2, unofficial].
*   **When to Use**: Ideal for reconstructing surfaces from point clouds where a true distance field is required for downstream tasks like ray marching or collision detection [S1, S2, unofficial].
*   **When Not to Use**: If only occupancy or binary segmentation is needed, simpler methods like [DeepSDF Neural Network](3051-deepsdf-neural-network.md) without Eikonal loss may suffice [S1, S2, unofficial].

## Background
The method originates from the need to train neural networks to output signed distance functions rather than just occupancy or density fields [S1, S2, unofficial]. The Eikonal equation $|\nabla \phi(x)| = 1$ is a fundamental property of signed distance functions [S1, S2, unofficial]. By enforcing this constraint during training, the network learns a geometrically consistent representation [S1, S2, unofficial]. This approach is used in methods like VolSDF, which converts SDF to volume density [S1, unofficial]. The implementation details provided in the sources are based on educational assignments and unofficial repositories [S1, S2, unofficial], while the Bias Network variant is described in the Point-NeuS paper [S3, unofficial].

## Sources
[S1] omkarchittar/Neural_Surfaces, https://github.com/omkarchittar/Neural_Surfaces, fetched 2026-10-26 (unofficial)
[S2] Fhz2346/Neural-Volume-Rendering-and-Surface-Rendering, https://github.com/Fhz2346/Neural-Volume-Rendering-and-Surface-Rendering, fetched 2026-10-27 (unofficial)
[S3] Point-NeuS: Point-Guided Neural Implicit Surface Reconstruction by Volume Rendering, https://doi.org/10.48550/arXiv.2310.07997, fetched 2026-10-30 (unofficial)
