# Progressive Training Scheme

Also called: Progressive depth sampling, progressive connection merging

## Overview
A training strategy for neural light field networks that incrementally increases neuron connections across training stages to prevent overfitting and improve detail reconstruction [S1, unofficial]. It solves the problem of point coordinate entanglement by starting with sparse, independent subnetworks to ensure point consistency, then progressively densifying connections to increase network capacity for high-quality rendering [S1, unofficial]. It sets itself apart from direct MLP training by enforcing an inductive bias for consistent scene geometry at the start, avoiding the overfitting and geometric inaccuracies typical of standard end-to-end optimization [S1, unofficial].

## How it works
**Inputs and Outputs:**
*   **Input:** A set of training images with known camera poses; a target scene geometry defined by near and far planes.
*   **Output:** A fully connected Multi-Layer Perceptron (MLP) with learned weights, capable of predicting color and density for any 3D point and viewing direction.

**Data Structures:**
The network is structured as a series of stages $k=0, \dots, K-1$.
*   **Subnetworks:** At stage $k$, the network consists of $N_k$ independent subnetworks. Each subnetwork processes coordinates for a specific subset of depth samples.
*   **Weights:** Weight matrices $W$ and bias vectors $b$ for each layer. During merging, new weights are initialized to zero.
*   **Depth Samples:** A discrete set of depth planes $d_i$ between the near and far planes.

**Parameters:**
*   **Stages:** Total $K=5$ stages (indexed 0 to 4) [S1, unofficial].
*   **Subnetworks at stage $k$:** $N_k = 16 \times 2^{-k}$ [S1, unofficial].
*   **Hidden width per subnetwork:** $W_k = 32 \times 2^k$ [S1, unofficial].
*   **Hidden depth:** 8 layers for ProLiF, 12 layers for ProLiF-Deep [S1, unofficial].
*   **Depth planes at stage $k$:** $D_k = 16 \times 2^k$ [S1, unofficial].
*   **Activation:** SIREN with $\omega_0 = 30$ [S1, unofficial].
*   **Optimizer:** Adam, initial learning rate $1 \times 10^{-4}$, decayed to $2.5 \times 10^{-6}$ via cosine scheduling [S1, unofficial].
*   **Iterations per stage:** $t$ steps (total 1M iterations across all stages) [S1, unofficial].
*   **Ray Sampling:** 16,384 rays for pixel fitting, 4,096 extra random rays for regularization per iteration [S1, unofficial].
*   **Input Normalization:** Coordinates normalized to NDC space [S1, unofficial].
*   **Weight Normalization:** Applied to stabilize training [S1, unofficial].

**Steps:**

1.  **Initialization (Stage 0):**
    *   Split the MLP into $D_0 = 16$ separated small subnetworks.
    *   Each subnetwork $j$ processes the coordinate of a point at a specific depth $d_j$ independently.
    *   Points at the same depth on different rays share the same subnetwork.
    *   Initialize weights using standard methods (e.g., SIREN initialization) [S1, unofficial].

2.  **Training Stage 0:**
    *   Train the independent subnetworks for $t$ steps.
    *   Loss function includes pixel fitting and regularization losses (e.g., color consistency, density consistency) [S1, unofficial].

3.  **Progressive Merging (Stage $k$ to $k+1$):**
    *   For $k$ from 0 to $K-2$:
        *   Identify pairs of neighboring subnetworks.
        *   Merge two neighboring subnetworks into one single fully connected network.
        *   **Feature Concatenation:** Concatenate the feature vectors of the two subnetworks.
        *   **Weight Merging:** Merge the weight parameters of the two subnetworks.
        *   **Zero Initialization:** Add zero values to the new weight parameters connecting the concatenated features to the next layer. This ensures the merged network initially behaves as two independent networks.
        *   **Channel Count:** Ensure the total channel number in the hidden layers of all points stays unchanged during merging [S1, unofficial].
        *   **Depth Subdivision:** Interpolate weights in subnetworks of neighboring depth values to process newly added depth samples. The number of sampled depth planes doubles from $16 \times 2^k$ to $16 \times 2^{k+1}$ [S1, unofficial].

4.  **Training Stage $k+1$:**
    *   Continue training with the new architecture for $t$ steps.
    *   Repeat steps 3-4 until a single fully connected network processes all input point coordinates (Stage 4).

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def progressive_training_scheme(num_stages=5, steps_per_stage=200000):
    # Initialize parameters
    num_depth_samples = 16
    hidden_width = 32
    hidden_depth = 8 # or 12 for ProLiF-Deep
    activation = SIREN(w0=30)
    optimizer = Adam(lr=1e-4)
    
    # Stage 0: Create independent subnetworks
    # Each subnetwork handles one depth sample
    subnetworks = []
    for i in range(num_depth_samples):
        # Create a small MLP for depth sample i
        # Input: 3D coordinate, Output: color/density
        subnetwork = create_mlp(input_dim=3, hidden_dim=hidden_width, 
                                output_dim=4, hidden_layers=hidden_depth, 
                                activation=activation)
        subnetworks.append(subnetwork)
        
    # Train Stage 0
    for step in range(steps_per_stage):
        rays = sample_rays(16384) # Pixel fitting rays
        reg_rays = sample_rays(4096) # Regularization rays
        loss = compute_loss(subnetworks, rays, reg_rays)
        optimizer.step(loss)
        
    # Progressive Merging and Training
    for k in range(1, num_stages):
        # Double the number of depth samples
        new_depth_samples = num_depth_samples * 2
        new_subnetworks = []
        
        # Merge pairs of subnetworks
        for i in range(0, len(subnetworks), 2):
            sub1 = subnetworks[i]
            sub2 = subnetworks[i+1]
            
            # Create merged subnetwork
            # Input: 3D coordinate
            # Hidden layers: width = hidden_width * 2 (concatenated features)
            merged_subnetwork = create_mlp(
                input_dim=3, 
                hidden_dim=hidden_width * 2, 
                output_dim=4, 
                hidden_layers=hidden_depth, 
                activation=activation
            )
            
            # Initialize weights from sub1 and sub2
            # Copy weights from sub1 to first half of merged weights
            # Copy weights from sub2 to second half of merged weights
            # Initialize new cross-connections to zero
            initialize_merged_weights(merged_subnetwork, sub1, sub2)
            
            new_subnetworks.append(merged_subnetwork)
            
        subnetworks = new_subnetworks
        num_depth_samples = new_depth_samples
        
        # Train merged subnetworks
        for step in range(steps_per_stage):
            rays = sample_rays(16384)
            reg_rays = sample_rays(4096)
            loss = compute_loss(subnetworks, rays, reg_rays)
            optimizer.step(loss)
            
    return subnetworks[0] # Final single fully connected network
```

**Edge Cases and Pitfalls:**
*   **Overfitting:** Direct training without the progressive scheme leads to overfitting and failure to reconstruct geometry [S1, unofficial].
*   **Artifacts:** Applying regularization losses without the progressive training scheme leaves observable artifacts in rendered images [S1, unofficial].
*   **Precision:** Zero initialization of new weights is critical to maintain the inductive bias during merging [S1, unofficial].

## Operating characteristics
*   **Time Complexity:** Training requires approximately 28 hours for ProLiF or 35 hours for ProLiF-Deep to complete 1M iterations on a single Nvidia 2080Ti GPU [S1, unofficial].
*   **Memory Complexity:** Model size is small (e.g., 8.0MB for LFN-SIREN baseline, ProLiF is comparable or smaller due to efficient representation) [S1, unofficial].
*   **Performance:** ProLiF outperforms NeRF in speed and image quality, and costs much less memory and time than Nex [S1, unofficial].
*   **Strengths:** Correctly reconstructs scene geometry; yields higher quality color and depth maps compared to ablated variants; handles high-frequency details via SIREN [S1, unofficial].
*   **Limitations:** Requires careful tuning of stage transitions; regularization is still necessary to avoid noise [S1, unofficial].
*   **When to use:** When training neural light field networks for view synthesis, especially when geometry consistency is critical and overfitting is a concern.
*   **When not to use:** For simple scenes where direct training suffices, or when computational resources are extremely limited and the overhead of multi-stage training is prohibitive.

## Background
*   **Origin:** Proposed in "Progressively-connected Light Field Network for Efficient View Synthesis" (Arxiv, 2022) [S1, unofficial].
*   **History:** Developed to address the inconsistency in neural light field representations caused by entanglement between point coordinates on a ray. The method builds on the idea of using independent subnetworks to enforce point consistency, gradually increasing capacity [S1, unofficial].
*   **Usage:** Used in the ProLiF and ProLiF-Deep architectures for efficient view synthesis [S1, unofficial].

## Sources
[S1] Progressively-connected Light Field Network for Efficient View Synthesis, https://doi.org/10.48550/arxiv.2207.04465, fetched 2026-10-27 (unofficial)
