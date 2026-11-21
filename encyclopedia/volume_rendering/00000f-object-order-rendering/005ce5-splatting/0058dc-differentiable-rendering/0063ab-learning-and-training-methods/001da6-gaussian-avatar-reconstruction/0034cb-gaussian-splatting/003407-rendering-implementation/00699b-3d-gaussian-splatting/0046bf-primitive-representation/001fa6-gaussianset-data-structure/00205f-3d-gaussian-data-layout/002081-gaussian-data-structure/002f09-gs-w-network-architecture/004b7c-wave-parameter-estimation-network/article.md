# Wave Parameter Estimation Network

Also called: Geometry-based Wave Parameter Fitting, Iterative Wave Parameter Optimization, UNet/MLP Wave Estimator

## Overview
The Wave Parameter Estimation Network (WPEN) estimates the constituent wave parameters (amplitude, angular frequency, and propagation direction) of a dynamic ocean from a time sequence of reconstructed geometric data, specifically surface normals and microfacet roughness [S1]. It distinguishes itself from direct regression methods by employing a differentiable ocean model as a physics-based loss function, allowing the network to learn by minimizing the geometric discrepancy between predicted and input ocean states [S1]. This approach enables self-supervised training where the network iteratively optimizes wave parameters to match real-world geometry reconstructed by preceding stages [S1].

## How it works
**Inputs and Outputs**
*   **Input:** A time sequence of reconstructed ocean geometry, consisting of surface normal maps ($N_t$) and roughness maps ($R_t$) for each time step $t$ [S1].
*   **Output:** A set of 60 constituent wave parameters. Each wave $i$ is defined by three parameters: amplitude $h_i$, angular frequency $\omega_i$, and propagation direction $\theta_i$ [S1].

**Data Structures**
The network architecture consists of two main components [S1]:
1.  **Feature Extractor:** A U-Net [41] that processes the input sequence of normal and roughness maps to extract spatial features.
2.  **Parameter Decoder:** A Multi-Layer Perceptron (MLP) that maps the extracted features to the final wave parameter vector [S1].

**Training Procedure**
The network is trained using a differentiable ocean model implemented in the Taichi framework [S1]. The training process involves the following steps:

1.  **Forward Pass:** The U-Net extracts features from the input geometry sequence ($N_t, R_t$). The MLP decodes these features into a batch of wave parameters $\{h_i, \omega_i, \theta_i\}_{i=1}^{60}$ [S1].
2.  **Differentiable Simulation:** The differentiable ocean model uses the predicted wave parameters to generate a synthetic sequence of ocean geometry ($\hat{N}_t, \hat{R}_t$) [S1].
3.  **Loss Calculation:** The synthetic geometry is compared against the input geometry using a combined loss function $L_{Geom}$ [S1].
    *   **Normal Loss ($L_{cos}$):** Measures the cosine similarity between the predicted normal $\hat{N}_t$ and the input normal $N_t$ [S1]:
        $$ L_{cos} = E_{N_t, \hat{N}_t} [(1 - \cos(N_t, \hat{N}_t))] $$
    *   **Roughness Loss ($L_2$):** Measures the Mean Squared Error (MSE) between the predicted roughness $\hat{R}_t$ and the input roughness $R_t$ [S1]:
        $$ L_2 = E_{R_t, \hat{R}_t} [\|R_t - \hat{R}_t\|_2^2] $$
    *   **Total Loss ($L_{Geom}$):** A weighted combination of the two losses [S1]:
        $$ L_{Geom} = \alpha L_2 + \beta L_{cos} $$
        where $\alpha$ and $\beta$ are hyperparameters balancing the importance of roughness and normal consistency [S1].
4.  **Optimization:** The network weights are updated via backpropagation through the differentiable ocean model to minimize $L_{Geom}$ [S1].

**Pseudocode**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def train_wave_parameter_network(input_geometry_sequence, differentiable_ocean_model, epochs=100):
    """
    Trains the WPEN to estimate wave parameters from geometry sequences.
    
    Args:
        input_geometry_sequence: List of tuples (N_t, R_t) for each time step t.
        differentiable_ocean_model: Differentiable renderer/simulator.
        epochs: Number of training epochs.
    """
    # Initialize network: U-Net for feature extraction, MLP for parameter decoding
    network = WaveParameterEstimationNetwork()
    
    # Hyperparameters for loss weighting
    alpha = 1.0  # Weight for roughness loss (Not given in sources, typical default)
    beta = 1.0   # Weight for normal loss (Not given in sources, typical default)
    
    optimizer = Adam(network.parameters())
    
    for epoch in range(epochs):
        total_loss = 0.0
        for t in range(len(input_geometry_sequence)):
            N_t, R_t = input_geometry_sequence[t]
            
            # 1. Extract features and predict wave parameters
            features = network.unet(N_t, R_t)
            wave_params = network.mlp(features)
            
            # 2. Generate synthetic geometry using differentiable ocean model
            # The model takes wave_params and returns predicted normal and roughness
            N_hat_t, R_hat_t = differentiable_ocean_model.simulate(wave_params)
            
            # 3. Compute losses
            # Cosine similarity loss for normals
            # cos(N_t, N_hat_t) is dot product of normalized vectors
            cos_sim = dot(normalize(N_t), normalize(N_hat_t))
            L_cos = mean(1.0 - cos_sim)
            
            # L2 (MSE) loss for roughness
            L_2 = mean((R_t - R_hat_t) ** 2)
            
            # Combined loss
            L_geom = alpha * L_2 + beta * L_cos
            
            # 4. Backpropagate and update
            L_geom.backward()
            optimizer.step()
            optimizer.zero_grad()
            
            total_loss += L_geom.item()
            
        print(f"Epoch {epoch}: Loss = {total_loss / len(input_geometry_sequence)}")
        
    return network
```

**Edge Cases and Pitfalls**
*   **Fuzzy Predictions:** Raw network predictions may be inaccurate in small areas or lack physical consistency [S1].
*   **Spectral Correction:** To mitigate fuzzy predictions, post-processing or constraints can be applied to enforce that the frequency fits a known spectral distribution and that frequency and amplitudes are negatively correlated [S1]. This correction is based on empirical wave spectrum properties (Eq. 2 in [S1], not explicitly defined in the provided text).
*   **Convergence:** The iterative optimization process relies on the differentiability of the ocean model; if the model is not fully differentiable or has discontinuities, convergence may fail [S1].

## Subtopics

### [Differentiable Ocean Inverse Modeling](13015-differentiable-ocean-inverse-modeling.md)
A two-step framework that uses WPEN as its second step. It first reconstructs geometry from monocular video and then uses WPEN to estimate wave parameters. Choose this when working with monocular video sequences and needing a self-supervised, differentiable end-to-end pipeline.

## Operating characteristics
*   **Time Complexity:** Training requires 100 epochs [S1]. Each epoch involves forward and backward passes through the U-Net, MLP, and the differentiable ocean model. The complexity is dominated by the differentiable simulation, which scales with the number of waves (60) and the resolution of the geometry maps [S1].
*   **Memory Complexity:** Depends on the U-Net architecture and the resolution of the input geometry sequences. The differentiable ocean model must store intermediate gradients for backpropagation [S1].
*   **Performance:** The method converges towards ground truth parameters, as demonstrated by the reduction in loss and visual comparison of rendered images [S1]. The accuracy is evaluated against ground truth wave parameters, with metrics including MSE for roughness and cosine similarity for normals [S1].
*   **Strengths:** Self-supervised; leverages physical constraints via the differentiable model; handles dynamic scenes [S1].
*   **Limitations:** Requires a differentiable ocean model; sensitive to the quality of input geometry (normals/roughness); may produce fuzzy results without spectral correction [S1].
*   **When to use:** When estimating wave parameters from reconstructed ocean geometry (normals/roughness) in a self-supervised manner.
*   **When not to use:** When direct regression from images is preferred without intermediate geometry reconstruction, or when a differentiable ocean model is not available.

## Background
The Wave Parameter Estimation Network is part of the "Dynamic ocean inverse modeling based on differentiable rendering" framework proposed by X. Xie, Y. Gao, F. Hou, et al. [S1]. The method was developed to reconstruct dynamic ocean scenes from real-world images and videos by estimating the underlying wave parameters that generate the observed geometry [S1]. The differentiable ocean model is implemented using the Taichi framework, which provides automatic differentiation capabilities [S1]. The network is trained after the geometry reconstruction stage (sub-net1 and sub-net2) has been trained for 200 epochs [S1].

## Sources
[S1] Dynamic ocean inverse modeling based on differentiable rendering, https://doi.org/10.1007/s41095-023-0338-4, fetched 2026-10-26
