# Progressive Training Scheme

Also called: Progressive depth sampling, progressive connection merging

## Overview
A training strategy for neural light field networks that incrementally increases neuron connections across training stages to prevent overfitting and improve detail reconstruction [S1, unofficial]. It solves the problem of point coordinate entanglement by starting with sparse, independent subnetworks to ensure point consistency, then progressively densifying connections to increase network capacity for high-quality rendering [S1, unofficial]. It sets itself apart from direct MLP training by enforcing an inductive bias for consistent scene geometry at the start, avoiding the overfitting and geometric inaccuracies typical of standard end-to-end optimization [S1, unofficial].

## How it works
**Inputs and Outputs:**
*   **Input:** Training images with known camera poses and a target scene geometry bounded by near and far planes [S1, unofficial].
*   **Output:** A fully connected Multi-Layer Perceptron (MLP) with learned weights, capable of predicting color and density for any 3D point and viewing direction [S1, unofficial].

**Data Structures:**
The network is structured as a series of $K=5$ stages indexed $k=0, \dots, 4$ [S1, unofficial].
*   **Subnetworks:** At stage $k$, the network consists of $N_k = 16 \times 2^{-k}$ independent subnetworks [S1, unofficial]. Each subnetwork $j$ processes the coordinate of a point at a specific depth $d_j$ independently [S1, unofficial]. Points at the same depth on different rays share the same subnetwork [S1, unofficial].
*   **Weights:** Weight matrices $W$ and bias vectors $b$ for each layer. During merging, new cross-connection weights are initialized to zero [S1, unofficial].
*   **Depth Samples:** A discrete set of depth planes $d_i$ between the near and far planes, with count $D_k = 16 \times 2^k$ at stage $k$ [S1, unofficial].

**Parameters:**
*   **Stages:** $K=5$ [S1, unofficial].
*   **Subnetworks at stage $k$:** $N_k = 16 \times 2^{-k}$ [S1, unofficial].
*   **Hidden width per subnetwork:** $W_k = 32 \times 2^k$ [S1, unofficial].
*   **Hidden depth:** 8 layers for ProLiF, 12 layers for ProLiF-Deep [S1, unofficial].
*   **Depth planes at stage $k$:** $D_k = 16 \times 2^k$ [S1, unofficial].
*   **Activation:** SIREN with $\omega_0 = 30$ [S1, unofficial].
*   **Optimizer:** Adam, initial learning rate $1 \times 10^{-4}$, decayed to $2.5 \times 10^{-6}$ via cosine scheduling [S1, unofficial].
*   **Iterations per stage:** $t = 200,000$ steps (total 1M iterations across all stages) [S1, unofficial].
*   **Ray Sampling:** 16,384 rays for pixel fitting and 4,096 extra randomly sampled rays for regularization per iteration [S1, unofficial].
*   **Input Normalization:** Coordinates normalized to NDC space [S1, unofficial].
*   **Weight Normalization:** Applied to stabilize training [S1, unofficial].

**Steps:**
1.  **Initialization (Stage 0):** Split the MLP into $D_0 = 16$ separated small subnetworks [S1, unofficial]. Each subnetwork $j$ processes the coordinate of a point at a specific depth $d_j$ independently [S1, unofficial]. Points at the same depth on different rays share the same subnetwork [S1, unofficial].
2.  **Training Stage 0:** Train the independent subnetworks for $t$ steps [S1, unofficial].
3.  **Progressive Merging (Stage $k$ to $k+1$):** Switch to the next stage by merging two neighboring subnetworks into one single fully connected network [S1, unofficial]. Concatenate features of the two subnetworks and add connections between adjacent layers of the merged network [S1, unofficial]. Construct new connections by merging weight parameters of the subnetworks and adding zero values to the new weight parameters [S1, unofficial]. The total channel number in the hidden layers of all points stays unchanged [S1, unofficial].
4.  **Depth Subdivision:** Progressively add more depth sampling values by interpolating weights in subnetworks of neighboring depth values for newly-added depth samples [S1, unofficial]. The number of sampled depth planes doubles to $16 \times 2^{k+1}$ [S1, unofficial].
5.  **Training Stage $k+1$:** Continue training with the architecture of the new stage for $t$ steps [S1, unofficial].
6.  **Repeat:** Repeat steps 3-5 until a single fully-connected network processes all input point coordinates [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def progressive_training_scheme(num_stages=5, steps_per_stage=200000):
    # Parameters
    init_depth_samples = 16
    hidden_width = 32
    hidden_depth = 8  # 12 for ProLiF-Deep
    activation_fn = lambda x: siren(x, w0=30)
    optimizer = Adam(lr=1e-4)

    # Stage 0: Initialize independent subnetworks
    subnetworks = []
    for _ in range(init_depth_samples):
        subnetworks.append(create_mlp(input_dim=3, hidden_dim=hidden_width, 
                                      output_dim=4, hidden_layers=hidden_depth, 
                                      activation=activation_fn))

    # Train Stage 0
    for _ in range(steps_per_stage):
        rays = sample_rays(16384)
        reg_rays = sample_rays(4096)
        loss = compute_loss(subnetworks, rays, reg_rays)
        optimizer.step(loss)

    # Progressive Merging and Depth Subdivision
    for k in range(1, num_stages):
        new_subnetworks = []
        # Merge pairs of neighboring subnetworks
        for i in range(0, len(subnetworks), 2):
            sub1 = subnetworks[i]
            sub2 = subnetworks[i+1]
            
            # Create merged subnetwork with doubled hidden width
            merged = create_mlp(input_dim=3, hidden_dim=hidden_width * 2, 
                                output_dim=4, hidden_layers=hidden_depth, 
                                activation=activation_fn)
            initialize_merged_weights(merged, sub1, sub2)
            new_subnetworks.append(merged)
            
            # Interpolate weights for the new depth sample between sub1 and sub2
            interp = create_mlp(input_dim=3, hidden_dim=hidden_width, 
                                output_dim=4, hidden_layers=hidden_depth, 
                                activation=activation_fn)
            interpolate_weights(interp, sub1, sub2)
            new_subnetworks.append(interp)
            
        subnetworks = new_subnetworks
        
        # Train merged and interpolated subnetworks
        for _ in range(steps_per_stage):
            rays = sample_rays(16384)
            reg_rays = sample_rays(4096)
            loss = compute_loss(subnetworks, rays, reg_rays)
            optimizer.step(loss)
            
    return subnetworks[0]  # Final single fully connected network

# Helper definitions
def siren(x, w0=30):
    return torch.sin(w0 * x)

class Adam:
    def __init__(self, lr=1e-4):
        self.lr = lr
    def step(self, loss):
        # Standard Adam update step
        pass

def create_mlp(input_dim, hidden_dim, output_dim, hidden_layers, activation):
    # Standard MLP initialization with weight normalization
    pass

def initialize_merged_weights(merged, sub1, sub2):
    # Copy weights from sub1 to first half of merged layers
    # Copy weights from sub2 to second half of merged layers
    # Initialize new cross-connections to zero
    pass

def interpolate_weights(interp, sub1, sub2):
    # Average weight parameters from sub1 and sub2
    pass

def sample_rays(count):
    # Return batch of rays for pixel fitting or regularization
    pass

def compute_loss(subnetworks, rays, reg_rays):
    # Compute pixel fitting and regularization losses
    pass
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

## Background
*   **Origin:** Proposed in "Progressively-connected Light Field Network for Efficient View Synthesis" (Arxiv, 2022) [S1, unofficial].
*   **History:** Developed to address the inconsistency in neural light field representations caused by entanglement between point coordinates on a ray [S1, unofficial]. The method builds on the idea of using independent subnetworks to enforce point consistency, gradually increasing capacity [S1, unofficial].
*   **Usage:** Used in the ProLiF and ProLiF-Deep architectures for efficient view synthesis [S1, unofficial].

## Sources
[S1] Progressively-connected Light Field Network for Efficient View Synthesis, https://doi.org/10.48550/arxiv.2207.04465, fetched 2026-10-27 (unofficial)
