# SDF Initialization via Overfitting

Also called: Inverted Sphere Pre-training, Neural Field Pre-training via Overfitting

## Overview
SDF Initialization via Overfitting is a learned initialization strategy for neural fields that do not accept position coordinates as direct MLP inputs. It addresses the difficulty of encoding a signed distance function (SDF) prior when the network architecture lacks the inductive bias of positional encoding or explicit coordinate inputs. Unlike methods that hardcode the SDF shape into weights, this approach initializes weights randomly and overfits the network to a target geometric prior (typically an inverted sphere) before beginning scene reconstruction [S1].

## How it works
The method consists of a two-phase process: pre-training to embed the geometric prior, followed by standard scene optimization.

**Inputs and Outputs**
*   **Input:** A target SDF shape definition (e.g., an inverted sphere of radius $R$ centered at the origin) and random initial network weights $W_0$.
*   **Output:** Optimized network weights $W_{init}$ that encode the inverted sphere SDF, ready for scene-specific fine-tuning.

**Data Structures**
*   **Network Weights ($W$):** Standard MLP parameters. No special bit-packing is required; weights are initialized from a standard distribution (e.g., Gaussian or Uniform) [S1].
*   **Target SDF Function ($S_{target}$):** A mathematical function defining the desired initial geometry. For an inverted sphere of radius $R$:
    $$ S_{target}(\mathbf{x}) = R - \|\mathbf{x}\| $$
    where $\mathbf{x}$ is a query point in the normalized space (typically $[-1, 1]^3$) [S1].

**Algorithm Steps**
1.  **Random Initialization:** Initialize the neural network weights $W$ randomly. The specific distribution is not detailed in the sources, but standard random initialization is implied [S1].
2.  **Pre-training (Overfitting):** Optimize the network weights $W$ to minimize the difference between the network's output and the target SDF values for a set of query points.
    *   **Loss Function:** Mean Squared Error (MSE) between network output and target SDF:
        $$ \mathcal{L}_{init}(W) = \frac{1}{N} \sum_{i=1}^{N} \left( \text{MLP}(\mathbf{x}_i; W) - S_{target}(\mathbf{x}_i) \right)^2 $$
        where $\mathbf{x}_i$ are query points sampled from the domain [S1].
    *   **Optimization:** Use a standard optimizer (e.g., Adam) to minimize $\mathcal{L}_{init}$.
    *   **Duration:** The network is trained for exactly 1000 epochs [S1].
3.  **Scene Reconstruction:** Use the weights $W$ obtained after pre-training as the starting point for the actual scene reconstruction task. The network is then fine-tuned on the target scene data (e.g., images, point clouds) [S1].

**Parameters**
*   **Pre-training Epochs:** 1000 [S1].
*   **Target Shape:** Inverted sphere [S1].

**Edge Cases and Pitfalls**
*   **Architecture Constraint:** This method is specifically required for networks that do *not* take position as a direct MLP input. If position is available, hardcoding the SDF into weights is preferred [S1].
*   **Precision:** The quality of the initialization depends on the convergence of the 1000-epoch pre-training. Insufficient epochs may result in a poor prior, leading to slower convergence or local minima during scene reconstruction [S1].

**Pseudocode**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def initialize_sdf_via_overfitting(network, target_radius=1.0, epochs=1000, lr=1e-3):
    """
    Initializes network weights by overfitting to an inverted sphere SDF.
    
    Args:
        network: Neural network model (MLP) with random weights.
        target_radius: Radius of the inverted sphere.
        epochs: Number of pre-training epochs.
        lr: Learning rate for the optimizer.
        
    Returns:
        network: Network with initialized weights.
    """
    # Step 1: Random initialization is assumed to be done externally
    # network = RandomInitialize() 
    
    optimizer = Adam(network.parameters(), lr=lr)
    
    # Define target SDF function: S(x) = R - ||x||
    def target_sdf(x):
        return target_radius - torch.norm(x, dim=-1)

    for epoch in range(epochs):
        # Sample query points x from the normalized domain [-1, 1]^3
        # [G] Assuming uniform sampling for simplicity; source does not specify sampling strategy
        x = torch.rand(1000, 3) * 2 - 1  # Shape: (N, 3)
        
        # Forward pass
        sdf_pred = network(x)
        
        # Compute target values
        sdf_target = target_sdf(x)
        
        # Compute loss
        loss = torch.mean((sdf_pred - sdf_target) ** 2)
        
        # Backward pass and optimization
        optimizer.zero_grad()
        loss.backward()
        optimizer.step()
        
    return network
```

## Subtopics

### SDF Initialization with Hotspot
This variant uses the Hotspot neural network [40] to initialize the SDF representation by overfitting to an input point cloud rather than a parametric shape. It offers two versions based on convergence: (1) a fully converged model trained for 10,000 iterations for precise SDF initialization, and (2) an unconverged model trained for 500 iterations to evaluate performance under imprecise SDF representation [S2]. This approach is suitable when a point cloud prior is available instead of a simple geometric shape.

## Operating characteristics
*   **Time Complexity:** The pre-training phase adds a fixed overhead of 1000 epochs [S1]. The cost is independent of the scene complexity but depends on the network size and query sampling density.
*   **Memory Complexity:** No additional memory structures are required beyond the network weights and temporary query points during pre-training.
*   **Strengths:** Effective for architectures lacking positional encoding or explicit coordinate inputs, where hardcoding the SDF prior is not feasible [S1].
*   **Limitations:** Requires a dedicated pre-training phase, increasing total training time. The quality of initialization is sensitive to the number of pre-training epochs and the sampling strategy for query points.
*   **When to Use:** When using neural field architectures that do not input position directly into the MLP and require a strong geometric prior to avoid catastrophic failure during optimization [S1].
*   **When Not to Use:** When the architecture supports direct position input, as hardcoding the SDF into weights is more efficient and effective [S1].

## Background
This method was introduced in the context of RING-NeRF, which rethinks inductive biases for versatile and efficient neural fields. It was developed to address the limitations of NeuS-facto and Neuralangelo, which hardcode the SDF inverted sphere into MLP weights. RING-NeRF's architecture does not input position directly, necessitating a different initialization strategy [S1]. The method was evaluated on Replica scenes, showing that without initialization, training often results in catastrophic failure or noisy reconstructions with lower PSNR metrics [S1].

## Sources
[S1] RING-NeRF : Rethinking Inductive Biases for Versatile and Efficient Neural Fields, https://doi.org/10.1007/978-3-031-72761-0_9, fetched 2026-10-30
[S2] DCCVT: Differentiable Clipped Centroidal Voronoi Tessellation, https://arxiv.org/abs/2601.13603v1, fetched 2026-10-26
