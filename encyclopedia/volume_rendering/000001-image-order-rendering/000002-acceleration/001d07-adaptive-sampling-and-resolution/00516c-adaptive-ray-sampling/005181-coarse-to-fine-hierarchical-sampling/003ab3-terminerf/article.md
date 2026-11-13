# TermiNeRF

Also called: Ray Termination Prediction, TermiNeRF Sampling Distribution Learning

## Overview
TermiNeRF is a method for efficient neural rendering that jointly trains a lightweight sampling network alongside a color network to predict the distribution of ideal sampling locations along a viewing ray [S1]. By concentrating samples where opacity is high and skipping free space, it significantly reduces the number of color network evaluations required per pixel [S1]. Unlike sibling methods that rely on depth maps or fixed spatial hierarchies, TermiNeRF conditions the sampling network on the density of a pretrained NeRF, allowing it to use the whole range of samples without explicit depth information, which improves quality in geometrically ambiguous regions [S2].

## How it works
TermiNeRF splits the traditional NeRF architecture into a **Sampling Network** and a **Shading (Color) Network**. The sampling network predicts weights for bins along the ray, which are then used to stochastically sample points for the shading network.

### Inputs and Outputs
*   **Input:** A viewing ray defined by origin $\mathbf{o}$ and direction $\mathbf{d}$, bounded by near $t_n$ and far $t_f$.
*   **Output:** A rendered pixel color $\mathbf{C}$ and, during training, gradients for both networks.

### Data Structures and Ray Parameterization
To avoid ambiguity in ray parameterization and uneven segment lengths, TermiNeRF uses a specific ray representation [S1]:

1.  **Line Segment:** A segment of constant length $\ell$ is defined by two points $A$ and $B$ such that $|\mathbf{B} - \mathbf{A}| = \ell$. The direction is $\vec{d} = \mathbf{B} - \mathbf{A}$.
2.  **Midpoint Constraint:** The midpoint of the segment is the closest point on the line to the scene origin: $\mathbf{O}_{mid} = \frac{\mathbf{A} + \mathbf{B}}{2}$.
3.  **Segment Length:** $\ell = 4.0$ [S1].
4.  **Sample Points:** $N$ points are placed along the segment using **centered-logarithmic sampling**. The points are $\{A, A + s_2 \cdot \vec{d}, \dots, A + s_{N-1} \cdot \vec{d}, B\}$.
    The offsets $s_i$ are formed by concatenating two sequences $l$ and $u$:
    $$ l_i = 1 - 2^{\frac{1-i}{N/2-1}}, \quad i \in \{1, \dots, N/2 - 1\} $$
    $$ u_j = 2^{\frac{j-N/2}{N/2-1}}, \quad j \in \{1, \dots, N/2\} $$
    The final set of sample positions $\mathbf{x}$ is derived from these offsets relative to $A$ and direction $\vec{d}$ [S1].

### Network Architectures
*   **Sampling Network:** An MLP with 8 hidden layers, 256 units per layer, and a skip connection to the fifth layer [S1]. It takes the ray parameters (derived from $\mathbf{o}, \mathbf{d}, A, B$) and outputs $N$ raw weights.
*   **Shading Network:** The standard NeRF MLP (coarse/fine or single network depending on implementation) that outputs RGB color and volume density $\sigma$ at queried points [S1, S2].

### Training Procedure
Training occurs in three phases to ensure stable convergence [S1]:

1.  **Pre-train Color Network:** Train the shading network using standard NeRF rendering with uniform sampling (e.g., 64 coarse + 128 fine samples) until convergence.
2.  **Train Sampling Network:**
    *   **Ground Truth Generation:** Evaluate the pretrained shading network at $N_{total}$ points (e.g., 196 points: 64 coarse + 128 fine) to obtain tuples $(z_i, \sigma_i)$.
    *   **Compute Weights:** Calculate ground truth weights $w_i$ for each bin using alpha compositing logic [S1]:
        $$ w_i = \exp\left(-\sum_{j=1}^{i-1} \sigma_j \delta_j\right) (1 - \exp(-\sigma_i \delta_i)) $$
        where $\delta_i = z_{i+1} - z_i$ [S1].
    *   **Normalization:** Normalize weights: $\hat{w}_i = \frac{w_i}{\sum_{j=0}^{N} w_j}$ [S1].
    *   **Distribution Matching (Max-Resampling):** If the ground truth points do not align with the $N$ bins defined by the sampling network's parameterization, perform max-resampling. For a new bin $b_j$ delimited by $[\hat{z}_j, \hat{z}_{j+1}]$, assign weight:
        $$ \hat{w}_j = \max \{ w_i \mid \hat{z}_j \le z_i \le \hat{z}_{j+1} \} \cup \{ w'_j, w'_{j+1} \} $$
        where $w'_j$ is the linear interpolation of weights from the old distribution at $\hat{z}_j$ [S1].
    *   **Blurring:** Apply Gaussian blur to the resampled weights to reduce noise. The Euclidean distance covered by the blur window is $d = \frac{K}{N_{samples}} \cdot \ell$. Typical parameters: window size $K=9$, sigma $\sigma=3.0$ [S1].
    *   **Loss:** Minimize MSE between the sampling network's output (normalized) and the blurred, normalized ground truth weights [S1].
3.  **Joint Fine-Tuning:** Alternate updates of the color and sampling networks.
    *   **Sampling Strategy:** Sample 128 points from the predicted distribution. Add 64 equidistantly sampled points along the whole ray between near and far $z$-values to ensure density values are acquired for unsampled regions (preventing the "blind spot" problem) [S1].
    *   **Optimization:** Use Adam optimizer. Learning rate $5 \cdot 10^{-4}$ for sampling network, $5 \cdot 10^{-5}$ for color network [S1].

### Rendering Pipeline
1.  **Discretize Ray:** Define the ray segment using the parameterization above.
2.  **Evaluate Sampling Network:** Input ray parameters to get $N$ weights $\hat{w}$.
3.  **Sample Points:** Draw $N_{render}$ points stochastically according to the distribution $\hat{w}$ [S1].
4.  **Evaluate Shading Network:** Query the shading network at the sampled points to get $(\mathbf{c}_i, \sigma_i)$.
5.  **Volumetric Rendering:** Aggregate color using standard alpha compositing [S1].

### Pseudocode

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import numpy as np

def generate_centered_logarithmic_samples(N, A, B):
    """
    Generate N sample points along segment A->B using centered-logarithmic sampling.
    A and B are vectors.
    """
    d = B - A
    half_N = N // 2
    
    # Generate l sequence
    i_indices = np.arange(1, half_N)
    l = 1 - 2**((1 - i_indices) / (half_N - 1))
    
    # Generate u sequence
    j_indices = np.arange(1, half_N + 1)
    u = 2**((j_indices - half_N) / (half_N - 1))
    
    # Concatenate l and u to get offsets s (excluding endpoints which are 0 and 1 relative to A->B)
    # The formula implies s_1...s_{N-1} are the internal points.
    # l has N/2 - 1 elements. u has N/2 elements. Total N-1 internal points.
    s = np.concatenate([l, u])
    
    # Points are A + s_k * d
    # Note: The paper defines points as {A, A+s2*d, ..., B}. 
    # s corresponds to the fractional distance from A.
    samples = A + s[:, np.newaxis] * d[np.newaxis, :]
    
    # Prepend A and append B if needed for bin boundaries, 
    # but for query points, we usually use the internal points or include endpoints.
    # The paper says "A, A+s2... B". Let's include endpoints for completeness of bins.
    all_points = np.vstack([A, samples, B])
    return all_points

def compute_ground_truth_weights(sigma, z_values):
    """
    Compute ground truth weights from density and z-values.
    """
    deltas = np.diff(z_values)
    # T_i = exp(-sum_{j=1}^{i-1} sigma_j * delta_j)
    # w_i = T_i * (1 - exp(-sigma_i * delta_i))
    
    # Cumulative sum of sigma * delta
    # sigma is shape (N,), deltas is shape (N-1,)
    # We need to align sigma with deltas. sigma[i] corresponds to delta[i] (between z[i] and z[i+1])
    # Standard NeRF: w_i = alpha_i * T_i
    # alpha_i = 1 - exp(-sigma_i * delta_i)
    # T_i = exp(-sum_{j=0}^{i-1} sigma_j * delta_j)
    
    alpha = 1 - np.exp(-sigma[:-1] * deltas) # Last sigma has no delta after it in this context usually, 
                                             # or we assume the last bin extends to infinity.
                                             # The paper formula sums j=1 to i-1.
    
    # Calculate Transmittance
    # T_0 = 1
    # T_i = T_{i-1} * exp(-sigma_{i-1} * delta_{i-1})
    transmittance = np.ones_like(alpha)
    for i in range(1, len(alpha)):
        transmittance[i] = transmittance[i-1] * np.exp(-sigma[i-1] * deltas[i-1])
        
    weights = transmittance * alpha
    
    # Normalize
    weights = weights / np.sum(weights)
    return weights

def max_resampling(old_z, old_weights, new_z_boundaries):
    """
    Distribute old weights to new bins using max-resampling.
    new_z_boundaries: array of N+1 boundaries defining N bins.
    """
    new_weights = np.zeros(len(new_z_boundaries) - 1)
    
    for j in range(len(new_weights)):
        z_min, z_max = new_z_boundaries[j], new_z_boundaries[j+1]
        
        # Find indices of old points within this bin
        mask = (old_z >= z_min) & (old_z <= z_max)
        candidates = old_weights[mask]
        
        if len(candidates) > 0:
            max_w = np.max(candidates)
        else:
            max_w = 0.0
            
        # Linear interpolation of weights at boundaries
        # w'_j is linear interpolation of weights from old distribution for z_min
        # This part is tricky: "w'_j is linear interpolation of weights from the old distribution for z_j"
        # Usually implies interpolating the weight function. 
        # If no points inside, we might just use 0 or interpolate boundary values if available.
        # For simplicity in implementation, if no points, we might skip or use 0.
        # The paper says: max { w_i | ... } U { w'_j, w'_{j+1} }
        # Let's assume if no points, we take 0 for the max part, and interpolate boundary weights if possible.
        
        # Interpolate weight at z_min
        # Find closest old points
        idx_min = np.searchsorted(old_z, z_min)
        if idx_min > 0 and idx_min < len(old_z):
            z1, z2 = old_z[idx_min-1], old_z[idx_min]
            w1, w2 = old_weights[idx_min-1], old_weights[idx_min]
            if z2 != z1:
                w_prime_min = w1 + (w2 - w1) * (z_min - z1) / (z2 - z1)
            else:
                w_prime_min = w1
        else:
            w_prime_min = 0.0
            
        idx_max = np.searchsorted(old_z, z_max)
        if idx_max > 0 and idx_max < len(old_z):
            z1, z2 = old_z[idx_max-1], old_z[idx_max]
            w1, w2 = old_weights[idx_max-1], old_weights[idx_max]
            if z2 != z1:
                w_prime_max = w1 + (w2 - w1) * (z_max - z1) / (z2 - z1)
            else:
                w_prime_max = w1
        else:
            w_prime_max = 0.0
            
        new_weights[j] = max(max_w, w_prime_min, w_prime_max)
        
    # Normalize
    new_weights = new_weights / np.sum(new_weights)
    return new_weights

def gaussian_blur_weights(weights, K=9, sigma=3.0, segment_length=4.0, N_samples=128):
    """
    Apply Gaussian blur to weights.
    """
    # Calculate Euclidean distance covered by window
    d = (K / N_samples) * segment_length
    
    # Simple 1D Gaussian kernel
    x = np.arange(-K//2 + 1, K//2 + 1)
    kernel = np.exp(-x**2 / (2 * sigma**2))
    kernel = kernel / np.sum(kernel)
    
    # Convolve
    blurred = np.convolve(weights, kernel, mode='same')
    return blurred

def train_terminerf():
    # 1. Pre-train Color Network
    color_net = train_nerf_uniform_sampling()
    
    # 2. Train Sampling Network
    sampling_net = init_sampling_network()
    
    # Define ray parameterization
    N_bins = 32 # Example
    A = np.array([-2.0, 0.0, 0.0])
    B = np.array([2.0, 0.0, 0.0])
    segment_length = 4.0
    
    # Get bin boundaries
    # Using centered logarithmic points as boundaries? 
    # The paper says "points as bins boundaries".
    # Let's assume the N bins are defined by the N+1 points generated.
    points = generate_centered_logarithmic_samples(N_bins + 1, A, B) # N+1 points for N bins
    z_values = np.linalg.norm(points - ray_origin, axis=1) # Distance from origin
    
    # Ground Truth Generation
    # Evaluate color net at 196 points (64 coarse + 128 fine)
    gt_z, gt_sigma = evaluate_color_net_fine_grained()
    gt_weights = compute_ground_truth_weights(gt_sigma, gt_z)
    
    # Distribution Matching
    # Map gt_z to the new bin boundaries defined by points
    # This step aligns the arbitrary NeRF samples to the structured TermiNeRF bins
    aligned_weights = max_resampling(gt_z, gt_weights, z_values)
    
    # Blurring
    blurred_weights = gaussian_blur_weights(aligned_weights, K=9, sigma=3.0, segment_length=segment_length, N_samples=N_bins)
    
    # Optimization Loop
    for epoch in range(num_epochs):
        # Forward pass through sampling network
        ray_params = get_ray_params(ray_o, ray_d)
        pred_weights = sampling_net(ray_params)
        pred_weights = pred_weights / np.sum(pred_weights) # Normalize
        
        # Loss
        loss = mse(pred_weights, blurred_weights)
        loss.backward()
        optimizer_sampling.step()
        
    # 3. Joint Fine-Tuning
    for epoch in range(num_fine_tune_epochs):
        # Sample 128 points from predicted distribution
        sampled_indices = sample_from_distribution(sampling_net(ray_params), 128)
        
        # Add 64 equidistant points
        equidistant_points = np.linspace(A, B, 64)
        
        all_points = np.concatenate([sampled_points, equidistant_points])
        
        # Evaluate color net
        colors, sigmas = color_net(all_points)
        
        # Render
        color = render_volumetric(colors, sigmas)
        
        # Loss
        loss = mse(color, target_color)
        loss.backward()
        
        # Alternate updates
        if epoch % 2 == 0:
            optimizer_sampling.step()
        else:
            optimizer_color.step()
```

### Edge Cases and Pitfalls
*   **Uneven Segment Lengths:** Using spherical intersection representations creates uneven segment lengths, increasing computational pressure on the network. TermiNeRF handles this by enforcing a constant length segment $\ell=4.0$ [S1].
*   **Blind Spots:** If the sampling network predicts low weights for a region, the color network never receives density values for that region, preventing the sampling network from learning geometry there. This is mitigated by adding 64 equidistant samples along the whole ray during joint fine-tuning [S1].
*   **Initialization:** Training both networks jointly from the start is inefficient because the sampling network receives arbitrary training signals. A pre-trained color network is required for initialization [S1, S2].
*   **Noise in Ground Truth:** Ground truth weights from NeRF are noisy. Gaussian blurring ($K=9, \sigma=3.0$) is essential to smooth these weights before training the sampling network [S1].

## Operating characteristics
*   **Speedup:** TermiNeRF achieves a 14x speedup relative to standard NeRF [S1]. In comparisons, TermiNeRF (32 samples) achieved similar or higher PSNR/LPIPS than NeRF (8+16 samples), representing a 7.32x speedup in forward passes [S1].
*   **Rendering Time:** Rendering time is reduced to $\le 1$ second per image [S1].
*   **Fine-Tuning Speed:** Scenes can be fine-tuned up to the penultimate step in only three minutes on a GTX 1080 [S1].
*   **Memory:** The sampling network has the same size and architecture as a NeRF network (8 hidden layers, 256 units), so memory overhead is minimal compared to the color network [S1].
*   **Strengths:**
    *   No depth map required, improving quality in geometrically ambiguous regions [S2].
    *   End-to-end trainable (after pre-training) [S1].
    *   Efficient fine-tuning for modified scenes (lighting/materials) [S1].
*   **Limitations:**
    *   Requires a pre-trained NeRF to initialize the color network [S2].
    *   Performance depends on the accuracy of the sampling network; false positives (sampling in empty space) do not break rendering but waste computation, while false negatives (missing geometry) cause artifacts [S1].

## Background
TermiNeRF was proposed by Martin Piala and Ronald Clark at Imperial College London [S1]. It was presented at the 2021 International Conference on 3D Vision (3DV) [S1]. The method addresses the slow rendering times of neural radiance fields (NeRF) by learning a direct mapping from camera rays to locations along the ray that are most likely to influence the pixel's final appearance [S1]. It builds on the concept of splitting the NeRF network into a sampling network and a shading network, a technique also explored in methods like AdaNeRF [S2].

## Sources
[S1] TermiNeRF: Ray Termination Prediction for Efficient Neural Rendering, https://doi.org/10.1109/3dv53792.2021.00118, fetched 2026-10-26
[S2] Recent Advances and Trends in Learning‐Based 3D Representations, https://doi.org/10.1111/cgf.70530, fetched 2026-10-26
