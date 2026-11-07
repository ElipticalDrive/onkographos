# Neural Transient Field (NeTF)


## Overview
Neural Transient Field (NeTF) is a neural modeling framework for Non-Line-of-Sight (NLOS) imaging that recovers the 5D transient function (spatial location and direction) using a Multi-Layer Perceptron (MLP) [S1]. It solves the inverse problem of reconstructing hidden 3D scenes from sparse, unbalanced Time-of-Flight (ToF) measurements captured by Single-Photon Avalanche Diodes (SPADs) [S1]. Unlike [NeRF](52-nerf.md), which parametrizes input data along linear rays, NeTF parametrizes input training data on spherical wavefronts, formulating the NLOS process under spherical coordinates analogous to volume rendering in Cartesian space [S1]. It is distinguished by its use of Markov Chain Monte Carlo (MCMC) to handle sparse sampling and a two-stage training strategy to address sampling bias [S1].

## How it works
**Inputs and Outputs:**
*   **Input:** Sparse set of detection spots on a relay wall, each associated with a measured transient histogram (time-of-flight data) [S1]. The input coordinates are parameterized on spherical wavefronts [S1].
*   **Output:** A 3D density volume implicitly represented by the MLP, which can be converted to a mesh via thresholding and Marching Cubes [S1].

**Data Structures:**
*   **MLP:** A Multi-Layer Perceptron that takes spatial coordinates and direction vectors as input and outputs density and color (or intensity) [S1]. The specific architecture (layer sizes, activation functions) is not detailed in the provided sources [G: Standard MLP architecture for neural fields, e.g., ReLU activations, positional encoding if implied by "neural modeling", but exact dimensions are Not given in the sources].
*   **Transient Field:** The 5D function mapping $(x, y, z, \theta, \phi)$ to transient intensity [S1].

**Algorithm Steps:**
1.  **Initialization:** Initialize the MLP parameters. [G: Standard random initialization for neural network weights].
2.  **Spherical Parameterization:** For each measurement spot on the relay wall, define the spherical wavefront geometry corresponding to the light transport path from the laser source, to the hidden object, to the detector spot [S1].
3.  **Sampling:** Use Markov Chain Monte Carlo (MCMC) to sample points within the 3D volume along the spherical wavefronts [S1]. This accounts for sparse and unbalanced sampling in the transient field [S1].
4.  **Network Evaluation:** Query the MLP with the sampled 3D coordinates and viewing directions to predict density $\sigma$ and color/intensity $c$ [S1].
5.  **Rendering/Integration:** Integrate the predicted density and color along the ray/wavefront to predict the transient response at the detector spot [S1]. The specific integration formula follows the volume rendering equation adapted for spherical coordinates, similar to the shared principles of the family [S1].
6.  **Loss Calculation:** Compute the loss between the predicted transient and the measured transient at every measuring spot on the relay wall [S1].
7.  **Optimization:** Update MLP parameters to minimize the loss [S1].
8.  **Reconstruction:** After training, extract the 3D shape by thresholding the density volume and applying Marching Cubes [S1].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import numpy as np
from scipy.interpolate import RegularGridInterpolator # Assuming standard interpolation if needed, but MCMC is key

def mcmc_sample(volume_bounds, num_samples, current_state):
    """
    Perform MCMC sampling within the volume bounds.
    Returns new state (3D coordinates).
    Note: Specific MCMC kernel (e.g., Metropolis-Hastings) details are Not given in the sources.
    """
    # [G] Placeholder for MCMC step: propose new position, accept/reject based on transient likelihood
    return current_state

def neural_transient_field_mlp(positions, directions, weights):
    """
    MLP that maps 3D position and direction to density and color.
    """
    # [G] Standard MLP forward pass: concat(position, direction), pass through layers, output sigma, c
    return density, color

def render_transient(positions, directions, weights):
    """
    Render transient response along spherical wavefronts.
    """
    # [G] Integrate density and color along the path, analogous to volume rendering equation
    # C(r) = sum(T_i * (1 - exp(-sigma_i * delta_i)) * c_i)
    return predicted_transient

def train_nelos(transient_measurements, wall_spots, volume_bounds, num_epochs):
    """
    Train NeTF using MCMC and two-stage training if specified.
    """
    weights = initialize_weights() # [G]
    
    for epoch in range(num_epochs):
        # 1. MCMC Sampling
        samples = []
        current_state = sample_initial_state(volume_bounds)
        for _ in range(num_samples_per_spot):
            current_state = mcmc_sample(volume_bounds, num_samples_per_spot, current_state)
            samples.append(current_state)
        
        # 2. Forward Pass
        predicted_transients = []
        for spot in wall_spots:
            # Get spherical wavefront geometry for this spot
            positions, directions = get_spherical_wavefront(spot, volume_bounds)
            # Sample along wavefront using MCMC
            sampled_positions, sampled_directions = mcmc_sample_wavefront(positions, directions, volume_bounds)
            
            density, color = neural_transient_field_mlp(sampled_positions, sampled_directions, weights)
            pred_transient = render_transient(sampled_positions, sampled_directions, weights)
            predicted_transients.append(pred_transient)
            
        # 3. Loss Calculation
        loss = compute_loss(predicted_transients, transient_measurements)
        
        # 4. Backpropagation
        weights = update_weights(weights, loss)
        
    return weights

def two_stage_training(transient_measurements, wall_spots, volume_bounds, num_epochs_stage1, num_epochs_stage2):
    """
    Two-stage training to address sampling bias.
    """
    # Stage 1: Initial training
    weights = train_nelos(transient_measurements, wall_spots, volume_bounds, num_epochs_stage1)
    
    # Calculate loss at every measuring spot
    losses = []
    for spot in wall_spots:
        # Render current prediction
        pred = render_transient_for_spot(spot, weights, volume_bounds)
        measured = get_measured_transient(spot)
        losses.append(compute_loss(pred, measured))
        
    # Normalize loss to form PDF
    pdf = normalize_losses(losses)
    
    # Resample detection spots based on PDF
    resampled_spots = resample_spots(wall_spots, pdf)
    
    # Stage 2: Retrain with resampled data
    # [G] The transient measurements are effectively re-weighted or the dataset is rebalanced
    weights = train_nelos(transient_measurements, resampled_spots, volume_bounds, num_epochs_stage2)
    
    return weights
```

**Parameters:**
*   **Sampling Spots:** e.g., 256x256 on the relay wall [S1].
*   **MCMC Samples:** Number of samples per spot is Not given in the sources.
*   **Learning Rate:** Not given in the sources.

**Edge Cases and Pitfalls:**
*   **Sparse Sampling:** Occluded points may be captured from very few viewpoints, leading to strong reconstruction artifacts if not handled [S1]. MCMC and two-stage training mitigate this [S1].
*   **Confocal vs. Non-Confocal:** The current form does not separately treat confocal and non-confocal setups [S1].
*   **Noise:** Learning-based techniques like NeTF incur much lower noise than SOTA methods, similar to NeRF [S1].
*   **Computational Cost:** Substantially more computationally expensive than previous optimization techniques [S1].

## Subtopics

### Two-stage Training
A training strategy to address sampling bias (undersampling) in NLOS reconstruction by resampling detection spots based on reconstruction loss [S1]. It consists of two stages: Stage 1 trains on all initial samples to get an initial reconstruction. Stage 2 calculates the loss at every measuring spot, normalizes it to a PDF, resamples spots (higher loss -> higher PDF -> denser sampling), and retrains on the new dataset [S1]. This recovers fine details (e.g., bunny ears) missing in one-stage training [S1].

## Operating characteristics
*   **Time Complexity:** Substantially more computationally expensive than previous optimization techniques (e.g., F-K, DLCT) [S1]. MCMC sampling adds overhead [S1].
*   **Memory Complexity:** Not given in the sources.
*   **Measured Performance:**
    *   **Confocal (Bunny):** MAE 2.65 cm (NeTF) vs. 2.38 cm (DLCT) and 2.43 cm (F-K) [S1].
    *   **Non-Confocal (Bunny):** MAE 7.45 cm (NeTF) vs. 3.77 cm (FBP) and 7.02 cm (BP) [S1].
    *   **Non-Confocal (Z):** MAE 0.60 cm (NeTF) vs. 0.46 cm (FBP) [S1].
*   **Strengths:**
    *   Produces sharper edges for front and back planes in semi-occluded scenes compared to Phasor Field and DLCT/F-K [S1].
    *   Separates different parts of a mesh, whereas DLCT exhibits adhesion [S1].
    *   Robust to silhouettes and semi-occlusions [S1].
    *   Lower noise than SOTA methods [S1].
    *   Works with very sparse sampling (e.g., 8x8 spots) [S1].
*   **Limitations:**
    *   Does not separately treat confocal and non-confocal setups [S1].
    *   High computational cost [S1].
    *   MAE may not fully reflect reconstruction quality (e.g., NeTF may have higher MAE but better silhouette preservation) [S1].
*   **When to Use:** When high-quality reconstruction of hidden scenes with complex topology, occlusions, or non-uniform albedo is required, and computational resources are available.
*   **When Not to Use:** When real-time performance is critical or computational resources are limited, or when confocal/non-confocal setups need separate optimization.

## Background
NeTF was introduced in the paper "Non-line-of-Sight Imaging via Neural Transient Fields" [S1]. It is inspired by Neural Radiance Fields (NeRF) but adapts the concept to NLOS imaging by using spherical wavefronts instead of linear rays [S1]. It addresses the challenges of sparse and unbalanced sampling in NLOS using MCMC and a two-stage training strategy [S1]. Tested on synthetic and real NLOS data including Bunny, Lucy, Indonesian, Bike, Letter Z, and semi-occluded scenes [S1].

## Sources
[S1] Non-line-of-Sight Imaging via Neural Transient Fields, https://doi.org/10.1109/tpami.2021.3076062, fetched 2026-10-26
