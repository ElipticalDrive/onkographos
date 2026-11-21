# TermiNeRF

Also called: Ray Termination Prediction, TermiNeRF Sampling Distribution Learning

## Overview
TermiNeRF is a method for efficient neural rendering that jointly trains a sampling network and a color network to predict the distribution of ideal sampling locations along a viewing ray [S1]. By concentrating samples where opacity is high and skipping free space, it significantly reduces the number of color network evaluations required per pixel [S1]. It sets itself apart from its siblings, [Coarse-to-Fine NeRF](3406-coarse-to-fine-hierarchical-sampling.md) and [NeRF-ID](8867-nerf-id.md), by using a single evaluation of a dedicated sampling network per ray to predict importance weights for a fixed set of bins, rather than relying on hierarchical coarse-to-fine refinement or implicit density-based adaptive step sizes [S1, S2].

## How it works
TermiNeRF splits the standard NeRF architecture into two components: a **Sampling Network** that predicts sample importance, and a **Shading (Color) Network** that renders the final color [S1, S2]. The method operates in three phases: pre-training the color network, training the sampling network, and joint fine-tuning [S1].

### Inputs and Outputs
*   **Input:** A viewing ray defined by origin $\mathbf{O}$ and direction $\mathbf{d}$.
*   **Output:** A rendered pixel color $C$ and, during training, gradients for both networks.

### Data Structures and Layout
1.  **Ray Parameterization:** Rays are discretized into a fixed set of bins using a "centered-logarithmic" sampling scheme to ensure constant segment lengths and avoid ambiguity in ray representation [S1].
    *   The ray segment is defined by two boundary points $A$ and $B$ such that $|\mathbf{B} - \mathbf{A}| = \ell$ [S1].
    *   The direction is $\vec{d} = \mathbf{B} - \mathbf{A}$ [S1].
    *   The midpoint of the segment is the closest point to the scene origin: $\mathbf{O}_{mid} = \frac{\mathbf{A}+\mathbf{B}}{2}$ [S1].
    *   The segment length $\ell$ is set to 4.0 [S1].
    *   Sample points are placed at:
        $$ \{ \mathbf{A}, \mathbf{A} + s_2 \cdot \vec{d}, \dots, \mathbf{A} + s_{N-1} \cdot \vec{d}, \mathbf{B} \} $$
        where the positions $s_i$ are formed by concatenating two sequences $l$ and $u$:
        $$ l_i = 1 - 2^{\frac{1-i}{N/2-1}}, \quad i \in \{1, \dots, N/2 - 1\} $$
        $$ u_j = 2^{\frac{j-N/2}{N/2-1}}, \quad j \in \{1, \dots, N/2\} $$
        The full set of sample positions is $\{l_1, \dots, l_{N/2-1}, u_1, \dots, u_{N/2}\}$ [S1].
2.  **Sampling Network:** An MLP with 8 hidden layers, 256 units per layer, and a skip connection to the fifth layer [S1]. It takes the ray parameters (derived from $A, B$) and outputs $N$ raw weights.
3.  **Shading Network:** The original NeRF color network (pre-trained) [S1].

### Numbered Steps

#### Phase 1: Pre-training
1.  Train the standard NeRF (color and density networks) using the original hierarchical sampling strategy (e.g., 64 coarse + 128 fine samples) until convergence [S1].

#### Phase 2: Training the Sampling Network
1.  **Generate Ground Truth Weights:** For each ray, evaluate the pre-trained fine NeRF network at $N_{gt}$ points (e.g., 196 points: 64 coarse + 128 fine + 64 additional) to obtain tuples $(z_i, \sigma_i)$ [S1].
2.  **Compute Raw Weights:** Calculate the volumetric rendering weight $w_i$ for each sample $i$ using the standard volume rendering equation [S1]:
    $$ w_i = \exp\left(-\sum_{j=1}^{i-1} \sigma_j \delta_j\right) (1 - \exp(-\sigma_i \delta_i)) $$
    where $\delta_i = z_{i+1} - z_i$ is the distance between neighboring samples [S1].
3.  **Normalize Weights:** Normalize the weights so they sum to 1:
    $$ \hat{w}_i = \frac{w_i}{\sum_{j=0}^{N} w_j} $$
4.  **Distribution Matching (Max-Resampling):** The ground truth weights are defined at arbitrary $z_i$ locations, but the sampling network predicts weights for fixed bins defined by the centered-logarithmic points. To align them:
    *   Define new bins $b_j$ delimited by $[\hat{z}_j, \hat{z}_{j+1}]$ based on the fixed sample positions [S1].
    *   Assign weight $\hat{w}_j$ to bin $b_j$ using max-resampling:
        $$ \hat{w}_j = \max \{ w_i \mid \hat{z}_j \le z_i \le \hat{z}_{j+1} \} \cup \{ w'_j, w'_{j+1} \} $$
        where $w'_j$ is the linear interpolation of weights from the old distribution at $\hat{z}_j$ [S1].
5.  **Gaussian Blurring:** Smooth the resampled weights to reduce noise from the NeRF estimation. Use a Gaussian blur window of size $K=9$ and $\sigma=3.0$ [S1]. The Euclidean distance covered by the window is $d = \frac{K}{N_{samples}} \cdot \ell$ [S1].
6.  **Normalize Blurred Weights:** Re-normalize the blurred weights to sum to 1 [S1].
7.  **Train Sampling Network:** Train the sampling network using Mean Squared Error (MSE) loss between its predicted normalized weights and the processed ground truth weights. Use Adam optimizer with learning rate $5 \cdot 10^{-4}$ [S1].

#### Phase 3: Joint Fine-Tuning
1.  **Sampling Strategy:** For each ray, sample $N_{pred}$ points (e.g., 128) stochastically according to the distribution predicted by the sampling network [S1].
2.  **Add Equidistant Samples:** To ensure the sampling network learns geometry in unsampled regions (free space), add $N_{eq}$ equidistantly sampled points (e.g., 64) along the whole ray between near and far z-values [S1].
3.  **Evaluate Color Network:** Evaluate the shading network at all sampled points (predicted + equidistant) to get $(\mathbf{c}_i, \sigma_i)$ [S1].
4.  **Render:** Aggregate colors using volumetric rendering [S1].
5.  **Update:** Alternate updates of the color and sampling networks using Adam optimizer. Learning rate for the sampling network is $5 \cdot 10^{-4}$ and for the color network is $5 \cdot 10^{-5}$ [S1].

### Pseudocode

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import math
import torch
import torch.nn as nn
import torch.optim as optim

class SamplingNetwork(nn.Module):
    """
    8 hidden layers, 256 units, skip connection to 5th layer.
    Input: Ray parameters (derived from A, B).
    Output: N raw weights.
    """
    def __init__(self, N):
        super().__init__()
        self.N = N
        # Architecture details inferred from [S1]
        self.layers = nn.ModuleList([
            nn.Linear(3, 256), # Assuming 3D input for simplicity, actual input is ray param
            nn.ReLU(),
            nn.Linear(256, 256),
            nn.ReLU(),
            nn.Linear(256, 256),
            nn.ReLU(),
            nn.Linear(256, 256),
            nn.ReLU(),
            nn.Linear(256, 256), # 5th layer
            nn.ReLU(),
            nn.Linear(256, 256),
            nn.ReLU(),
            nn.Linear(256, 256),
            nn.ReLU(),
            nn.Linear(256, 256),
            nn.ReLU(),
            nn.Linear(256, 256),
            nn.ReLU(),
            nn.Linear(256, N)
        ])
        self.skip_layer_idx = 4 # 0-indexed, corresponds to 5th layer

    def forward(self, ray_input):
        x = ray_input
        for i, layer in enumerate(self.layers):
            x = layer(x)
            if i == self.skip_layer_idx:
                skip = x
            if i > self.skip_layer_idx:
                x = x + skip
        return x

def get_centered_logarithmic_samples(N, A, B):
    """
    Generates N sample positions along segment A->B.
    Returns list of positions s_i relative to A.
    """
    d = B - A
    samples = []
    # l_i for i in 1..N/2-1
    half = N // 2
    l_vals = []
    for i in range(1, half):
        val = 1 - 2**((1 - i) / (half - 1))
        l_vals.append(val)
    
    # u_j for j in 1..N/2
    u_vals = []
    for j in range(1, half + 1):
        val = 2**((j - half) / (half - 1))
        u_vals.append(val)
        
    # Concatenate l and u
    all_s = l_vals + u_vals
    
    # Return points A + s_i * d
    points = [A + s * d for s in all_s]
    return points

def gaussian_blur(weights, K=9, sigma=3.0):
    """
    Applies Gaussian blur to 1D weight array.
    """
    # Implementation of 1D Gaussian convolution
    kernel_size = K
    kernel = torch.exp(-torch.arange(kernel_size) ** 2 / (2 * sigma ** 2))
    kernel = kernel / kernel.sum()
    
    # Pad and convolve
    padded = torch.nn.functional.pad(weights, (kernel_size//2, kernel_size//2), mode='reflect')
    blurred = torch.nn.functional.conv1d(padded.unsqueeze(0).unsqueeze(0), kernel.unsqueeze(0).unsqueeze(0)).squeeze()
    return blurred

def max_resample(old_z, old_w, new_z):
    """
    Resamples old weights to new bin boundaries defined by new_z.
    new_z defines the boundaries of the bins.
    """
    new_w = []
    for j in range(len(new_z) - 1):
        z_start = new_z[j]
        z_end = new_z[j+1]
        
        # Find max weight in range
        mask = (old_z >= z_start) & (old_z <= z_end)
        local_max = 0.0
        if torch.any(mask):
            local_max = torch.max(old_w[mask])
            
        # Interpolate weights at boundaries
        w_start = interpolate_weight(old_z, old_w, z_start)
        w_end = interpolate_weight(old_z, old_w, z_end)
        
        # Max of local max and interpolated boundaries
        w_j = max(local_max, w_start, w_end)
        new_w.append(w_j)
        
    return torch.tensor(new_w)

def interpolate_weight(z, w, z_target):
    """
    Linear interpolation of weight at z_target given z, w arrays.
    """
    # Find indices surrounding z_target
    # This is a simplified placeholder for the actual interpolation logic
    # In practice, use torch.searchsorted or similar
    pass

def train_sampling_network(color_net, sampling_net, dataloader, N_samples=128, N_eq=64):
    """
    Phase 2: Train Sampling Network against pre-trained Color Net.
    """
    optimizer = optim.Adam(sampling_net.parameters(), lr=5e-4)
    sampling_net.train()
    color_net.eval()
    
    for ray, gt_color in dataloader:
        # 1. Get ground truth weights from Color Net
        # Evaluate fine NeRF at 196 points (64 coarse + 128 fine + 64 extra)
        z_gt, sigma_gt = evaluate_fine_nerf(color_net, ray, num_points=196)
        
        # 2. Compute raw weights
        delta = z_gt[1:] - z_gt[:-1]
        transmittance = torch.exp(-torch.cumsum(sigma_gt[:-1] * delta, dim=0))
        raw_w = transmittance * (1 - torch.exp(-sigma_gt[:-1] * delta))
        
        # 3. Normalize
        raw_w = raw_w / raw_w.sum()
        
        # 4. Define fixed bins using centered-logarithmic sampling
        # Assume A and B are derived from ray parameters
        A, B = get_ray_boundaries(ray)
        fixed_points = get_centered_logarithmic_samples(N_samples, A, B)
        new_z = [A + (B-A) * s for s in fixed_points] # Approximation, actual bins are between points
        
        # 5. Max-Resample
        # Note: The paper describes resampling to match the bins defined by the fixed points
        # We need to map old_z to new bin centers/boundaries
        new_w = max_resample(z_gt, raw_w, new_z)
        
        # 6. Blur
        new_w = gaussian_blur(new_w, K=9, sigma=3.0)
        
        # 7. Normalize again
        new_w = new_w / new_w.sum()
        
        # 8. Forward pass through Sampling Network
        ray_input = get_ray_input(ray) # Extract A, B, etc.
        pred_raw_w = sampling_net(ray_input)
        pred_w = pred_raw_w / pred_raw_w.sum()
        
        # 9. Loss
        loss = nn.MSELoss()(pred_w, new_w)
        
        optimizer.zero_grad()
        loss.backward()
        optimizer.step()

def render_terminerf(color_net, sampling_net, ray, N_pred=128, N_eq=64):
    """
    Phase 3: Joint Fine-Tuning / Inference.
    """
    sampling_net.eval()
    color_net.eval()
    
    # 1. Predict distribution
    ray_input = get_ray_input(ray)
    pred_raw_w = sampling_net(ray_input)
    pred_w = pred_raw_w / pred_raw_w.sum()
    
    # 2. Sample N_pred points stochastically from pred_w
    # Map probabilities to positions along the ray
    sampled_z = sample_from_distribution(pred_w, N_pred)
    
    # 3. Add N_eq equidistant points
    A, B = get_ray_boundaries(ray)
    eq_z = torch.linspace(A, B, N_eq)
    
    # 4. Combine and sort
    all_z = torch.cat([sampled_z, eq_z]).sort()[0]
    
    # 5. Evaluate Color Network
    # Get sigma and color at all_z
    sigma, color = evaluate_color_net(color_net, ray, all_z)
    
    # 6. Volumetric Rendering
    delta = all_z[1:] - all_z[:-1]
    transmittance = torch.exp(-torch.cumsum(sigma[:-1] * delta, dim=0))
    final_color = (transmittance * color[:-1] * delta).sum() # Simplified aggregation
    
    return final_color
```

### Parameters
*   **Segment Length ($\ell$):** 4.0 [S1].
*   **Blur Window Size ($K$):** 9 [S1].
*   **Blur Sigma ($\sigma$):** 3.0 [S1].
*   **Sampling Network Learning Rate:** $5 \cdot 10^{-4}$ [S1].
*   **Color Network Learning Rate:** $5 \cdot 10^{-5}$ [S1].
*   **Samples per Ray (Training):** 128 predicted + 64 equidistant [S1].
*   **Samples per Ray (Inference):** Variable, e.g., 32 [S1].

### Edge Cases and Pitfalls
*   **Uneven Segment Lengths:** Using sphere intersection representations creates uneven segment lengths, which puts computational pressure on the network. This is mitigated by enforcing a constant length segment $\ell$ [S1].
*   **Arbitrary Training Signals:** Training both networks jointly from the start is inefficient because the sampling network receives arbitrary signals from an untrained color network. The color network must be pre-trained first [S1].
*   **Free Space Learning:** If only predicted samples are used, the sampling network may never learn geometry in unsampled regions (free space). Adding equidistant samples ensures density values are acquired for these regions [S1].
*   **False Positives:** A false-positive sampling network misprediction (predicting a sample in free space) poses no issue for the color network, as the density will be low [S1].

## Operating characteristics
*   **Speed:** Rendering time reduced to $\le 1$ second for an image [S1]. Achieves a 14x speedup relative to standard NeRF [S1].
*   **Quality:** Achieves similar or higher PSNR/LPIPS than NeRF with significantly fewer forward passes (e.g., 7.32x speedup for TermiNeRF 32 vs NeRF 8+16) [S1].
*   **Fine-Tuning:** Scenes can be fine-tuned up to the penultimate step in only three minutes on a GTX1080 [S1].
*   **Memory:** Requires storing the sampling network in addition to the color network.
*   **Limitations:** Requires a pre-trained NeRF to initialize the color network for best results [S2].

## Background
TermiNeRF was introduced by Martin Piala and Ronald Clark from Imperial College London in 2021 [S1]. It addresses the slow rendering times of neural radiance fields (NeRF) by learning a direct mapping from camera rays to locations along the ray that are most likely to influence the pixel's final appearance [S1]. The method is cited in later surveys on learning-based 3D representations as a key example of splitting NeRF into a sampling network and a shading network [S2].

## Sources
[S1] TermiNeRF: Ray Termination Prediction for Efficient Neural Rendering, https://doi.org/10.1109/3dv53792.2021.00118, fetched 2026-10-26
[S2] Recent Advances and Trends in Learning‐Based 3D Representations, https://doi.org/10.1111/cgf.70530, fetched 2026-10-26
