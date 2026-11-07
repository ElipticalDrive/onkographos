# TermiNeRF

Also called: Ray Termination Prediction, TermiNeRF Sampling Distribution Learning

## Overview
TermiNeRF is a method for efficient neural rendering that jointly trains a lightweight sampling network alongside a color (shading) network to predict the distribution of ideal sampling locations along a viewing ray [S1]. By concentrating samples where opacity is high and skipping free space, it significantly reduces the number of color network evaluations required per pixel [S1]. Unlike sibling methods that rely on depth maps or spatial data structures (like quadtrees or occupancy grids) to guide sampling, TermiNeRF predicts sample weights directly from ray parameters using a single network evaluation per ray, making it robust in geometrically ambiguous regions [S2]. It builds on [Adaptive Ray Sampling](8870-adaptive-ray-sampling.md) by learning the probability distribution $P(t)$ explicitly rather than deriving it from a coarse pass or heuristic.

## How it works
### Inputs and Outputs
*   **Inputs:** Camera ray defined by origin $\mathbf{O}$ and direction $\mathbf{d}$; pre-trained color network $F_{color}(\mathbf{x}, \mathbf{d}) \to (\mathbf{c}, \sigma)$; sampling network $F_{sample}(\mathbf{O}, \mathbf{d}) \to \{\hat{w}_i\}_{i=1}^N$.
*   **Outputs:** Rendered pixel color $\mathbf{C}$; optimized sampling network weights.

### Data Structures
**Ray Parameterization:**
To avoid ambiguity in ray representation, TermiNeRF enforces a constant-length line segment $[A, B]$ with length $\ell = 4.0$ [S1].
*   The segment midpoint is the closest point on the line to the scene origin: $\mathbf{O}_{mid} = \frac{\mathbf{A} + \mathbf{B}}{2} = \text{proj}_{\mathbf{AB}}(\mathbf{0})$ [S1].
*   The direction is $\mathbf{d} = \mathbf{B} - \mathbf{A}$ [S1].
*   Points $A$ and $B$ must lie outside the rendered object [S1].

**Sample Locations (Bins):**
The ray is discretized into $N$ bins using "centered-logarithmic sampling" [S1]. The sample points are:
$$ \{ \mathbf{A}, \mathbf{A} + s_2 \cdot \mathbf{d}, \dots, \mathbf{A} + s_{N-1} \cdot \mathbf{d}, \mathbf{B} \} $$
where the scalar coefficients $s_i$ are formed by concatenating two sequences $l$ and $u$:
$$ l_i = 1 - 2^{\frac{1-i}{N/2-1}}, \quad i \in \{1, \dots, N/2 - 1\} $$
$$ u_j = 2^{\frac{j-N/2}{N/2-1}}, \quad j \in \{1, \dots, N/2\} $$
The last bin extends from the last boundary point to infinity [S1].

**Network Architecture:**
*   **Sampling Network:** 8 hidden layers with 256 units each, with a skip connection to the fifth layer [S1].
*   **Color Network:** Standard NeRF architecture (pre-trained) [S1].

### Training Procedure
The training occurs in three phases to ensure stable convergence [S1]:

1.  **Pre-training:** Train the color network $F_{color}$ on the dataset until convergence [S1].
2.  **Sampling Network Training:** Train $F_{sample}$ to predict weights that match the importance distribution derived from the fine color network [S1].
3.  **Joint Fine-tuning:** Alternately update both networks to adapt to scene modifications or refine geometry [S1].

#### Step 1: Ground Truth Weight Generation
For each ray, evaluate the fine color network at $64 + 128 = 196$ equidistant points to obtain tuples $(z_i, \sigma_i)$, where $z_i$ is the distance from the ray origin [S1]. Compute the ground truth rendering weights $w_i$ for each sample $i$ using the standard volume rendering opacity formula [S1]:
$$ w_i = \exp\left(-\sum_{j=1}^{i-1} \sigma_j \delta_j\right) (1 - \exp(-\sigma_i \delta_i)) $$
where $\delta_i = z_{i+1} - z_i$ [S1].

#### Step 2: Ground Truth Processing (Blurring and Distribution Matching)
The raw weights $w_i$ are noisy and defined on a fixed grid, not the adaptive bins predicted by the sampling network. They must be processed to create supervision targets $\hat{w}_j$ [S1].

1.  **Distribution Matching (Max-Resampling):**
    Because the sampling network predicts weights for specific bin boundaries $\hat{z}_j$, we must map the old weights $w_i$ (at $z_i$) to the new bins $b_j$ (delimited by $[\hat{z}_j, \hat{z}_{j+1}]$). To preserve peaks and handle varying local density, use max-resampling [S1]:
    $$ \hat{w}_j = \max \left( \{ w_i \mid \hat{z}_j \le z_i \le \hat{z}_{j+1} \} \cup \{ w'_j, w'_{j+1} \} \right) $$
    where $w'_j$ is the linear interpolation of weights from the old distribution at $\hat{z}_j$ [S1].
    *Note:* If the number of elements in the blur window varies due to local density, this step ensures samples are placed evenly before blurring [S1].

2.  **Gaussian Blurring:**
    Apply a Gaussian blur to the resampled weights to smooth noise. The blur window size $K$ and standard deviation $\sigma_{blur}$ are fixed hyperparameters [S1].
    *   Typical values: $K=9$, $\sigma_{blur}=3.0$ [S1].
    *   The Euclidean distance covered by the window is $d = \frac{K}{N_{samples}} \cdot \ell$. For the Lego dataset with $N_{samples}=128$ and $\ell=4.0$, $d \approx 0.28$ [S1].

3.  **Normalization:**
    Normalize the resulting weights so they sum to 1:
    $$ \hat{w}_i = \frac{\hat{w}_i}{\sum_{j=0}^{N} \hat{w}_j} $$
    [S1].

#### Step 3: Sampling Network Update
The sampling network takes the ray parameters $(\mathbf{O}, \mathbf{d})$ as input and outputs $N$ raw scores. These are normalized to probabilities $p_i$ and compared against the processed ground truth $\hat{w}_i$ using Mean Squared Error (MSE) loss [S1]:
$$ \mathcal{L}_{sample} = \sum_{i=1}^N (p_i - \hat{w}_i)^2 $$
Optimize using Adam with learning rate $5 \cdot 10^{-4}$ [S1].

#### Step 4: Joint Fine-tuning
During joint training, alternate updates of the color and sampling networks [S1].
*   **Sampling:** Draw 128 points stochastically from the distribution predicted by the sampling network [S1].
*   **Regularization:** Add 64 equidistantly sampled points along the whole ray between near and far $z$-values to ensure density values are acquired for unsampled regions (preventing the sampling network from ignoring areas it previously deemed low-weight) [S1].
*   **Optimization:** Use Adam with learning rate $5 \cdot 10^{-5}$ for the color network and $5 \cdot 10^{-4}$ for the sampling network [S1].

### Pseudocode

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import numpy as np

def generate_centered_logarithmic_samples(N, A, B):
    """
    Generates N-2 internal sample points between A and B using centered-logarithmic spacing.
    Returns array of points including A and B.
    """
    d = B - A
    # Calculate coefficients l and u
    # l_i for i in 1..N/2-1
    # u_j for j in 1..N/2
    # Note: The formula uses 1-based indexing for description, 
    # we map to 0-based array indices for implementation.
    
    half_N = N // 2
    
    # l sequence: i from 1 to half_N - 1
    l_indices = np.arange(1, half_N)
    l_coeffs = 1 - 2**((1 - l_indices) / (half_N - 1))
    
    # u sequence: j from 1 to half_N
    u_indices = np.arange(1, half_N + 1)
    u_coeffs = 2**((u_indices - half_N) / (half_N - 1))
    
    # Concatenate l and u
    s_coeffs = np.concatenate([l_coeffs, u_coeffs])
    
    # Generate points: A + s_k * d
    # The full set is A, A+s_2*d, ..., A+s_{N-1}*d, B
    # s_coeffs corresponds to s_2 ... s_{N-1}
    internal_points = A + s_coeffs.reshape(-1, 1) * d
    
    # Combine with boundaries
    samples = np.vstack([A.reshape(1, -1), internal_points, B.reshape(1, -1)])
    return samples

def compute_ground_truth_weights(z_values, sigma_values):
    """
    Computes rendering weights w_i from optical depth sigma and distances z.
    """
    # delta_i = z_{i+1} - z_i
    deltas = np.diff(z_values)
    
    # Transmittance T_i = exp(-sum_{j=1}^{i-1} sigma_j * delta_j)
    # We compute cumulative sum of sigma * delta
    sigma_delta = sigma_values[:-1] * deltas # sigma_i * delta_i for i=1..N-1
    # Note: The formula sums j=1 to i-1. 
    # For i=1, sum is empty -> T=1.
    # For i=2, sum is sigma_1 * delta_1.
    
    # Cumulative sum of sigma * delta
    cum_sum = np.cumsum(sigma_delta)
    # Prepend 0 for the first term (empty sum)
    cum_sum = np.concatenate([[0], cum_sum])
    
    transmittance = np.exp(-cum_sum)
    
    # Weight w_i = T_i * (1 - exp(-sigma_i * delta_i))
    # sigma_i * delta_i for i=1..N-1. Last point has no delta in this context 
    # or delta is effectively 0/infinity depending on boundary. 
    # NeRF usually evaluates N points. The last weight is often handled 
    # by the boundary condition or assumed to be 0 contribution if outside.
    # Based on formula: w_i = exp(...) * (1 - exp(-sigma_i * delta_i))
    
    # sigma_i * delta_i for i=0..N-2 (0-indexed)
    sigma_delta_i = sigma_values[:-1] * deltas
    
    weights = transmittance[:-1] * (1 - np.exp(-sigma_delta_i))
    
    # The last point (index N-1) usually doesn't have a delta in the standard 
    # discrete integration unless defined. In TermiNeRF context, 
    # we have 196 tuples. The formula applies to the bins.
    # We return weights for the bins defined by the samples.
    return weights

def max_resampling(old_z, old_w, new_z_boundaries):
    """
    Maps old weights to new bins using max-resampling.
    new_z_boundaries: array of size N+1 defining N bins [z_j, z_{j+1}]
    """
    N_bins = len(new_z_boundaries) - 1
    new_w = np.zeros(N_bins)
    
    for j in range(N_bins):
        z_low = new_z_boundaries[j]
        z_high = new_z_boundaries[j+1]
        
        # Find old weights within the bin
        mask = (old_z >= z_low) & (old_z <= z_high)
        candidates = old_w[mask]
        
        # Linear interpolation of weights at boundaries
        # w'_j is interpolation at z_low
        # w'_{j+1} is interpolation at z_high
        # Assuming old_z is sorted
        w_prime_low = np.interp(z_low, old_z, old_w)
        w_prime_high = np.interp(z_high, old_z, old_w)
        
        # Max of candidates and interpolated boundaries
        all_candidates = np.concatenate([candidates, [w_prime_low, w_prime_high]])
        new_w[j] = np.max(all_candidates)
        
    return new_w

def train_sampling_network(color_net, sample_net, ray_o, ray_d, N_samples=128):
    """
    Single step of training the sampling network.
    """
    # 1. Get Ground Truth from Color Net
    # Evaluate fine color net at 196 equidistant points
    z_gt = np.linspace(ray_o, ray_o + 4.0 * ray_d, 196) # Assuming length 4.0
    # Get sigma from color_net
    sigma_gt = color_net(z_gt, ray_d) 
    
    # Compute raw weights
    w_gt = compute_ground_truth_weights(z_gt, sigma_gt)
    
    # 2. Define Target Bins using Centered-Logarithmic Sampling
    # A and B are boundaries. For a ray of length 4.0 centered at origin projection:
    # This requires solving for A and B such that |B-A|=4.0 and midpoint is proj.
    # Simplified: Assume A and B are known from ray parameterization logic.
    A = ... # Derived from ray O, d and scene bounds
    B = ... 
    
    sample_points = generate_centered_logarithmic_samples(N_samples, A, B)
    new_z_boundaries = sample_points # These define the bins
    
    # 3. Process Ground Truth
    # Max-resampling
    w_resampled = max_resampling(z_gt, w_gt, new_z_boundaries)
    
    # Gaussian Blur (Simplified: convolution with Gaussian kernel)
    # K=9, sigma=3.0
    kernel = np.exp(-np.arange(-4, 5)**2 / (2 * 3.0**2))
    kernel /= kernel.sum()
    w_blurred = np.convolve(w_resampled, kernel, mode='same')
    
    # Normalize
    w_target = w_blurred / w_blurred.sum()
    
    # 4. Sample Network Forward Pass
    # Input: Ray parameters (O, d) encoded appropriately
    raw_output = sample_net(ray_o, ray_d)
    
    # Normalize output to probability distribution
    p_pred = raw_output / raw_output.sum()
    
    # 5. Loss and Update
    loss = np.mean((p_pred - w_target)**2)
    # Backpropagate loss to sample_net
    sample_net.update(loss)
    
    return loss

def render_with_terminerf(color_net, sample_net, ray_o, ray_d, N_samples=128):
    """
    Rendering step.
    """
    # 1. Predict distribution
    raw_weights = sample_net(ray_o, ray_d)
    probs = raw_weights / raw_weights.sum()
    
    # 2. Stochastic Sampling
    # Draw N_samples points from the predicted distribution
    # In practice, this might involve inverse transform sampling or 
    # simply selecting indices based on cumulative probs.
    # TermiNeRF draws points stochastically according to the distribution.
    indices = np.random.choice(N_samples, size=N_samples, p=probs)
    
    # 3. Evaluate Color Net at sampled locations
    # Locations are determined by the bin boundaries associated with the indices
    # Or more precisely, the samples are drawn from the distribution defined by the bins.
    # The pseudocode above defines bins. We sample points *within* or *at* these bins?
    # "Ray points drawn stochastically according to this distribution are then 
    # separately evaluated by the color network."
    # Usually, this means evaluating the color net at the sample points 
    # corresponding to the high-weight bins.
    
    # Simplified: Evaluate at the bin centers or boundaries of the selected bins
    sample_points = generate_centered_logarithmic_samples(N_samples, A, B)
    selected_points = sample_points[indices]
    
    colors = color_net(selected_points, ray_d)
    
    # 4. Volumetric Rendering
    # Aggregate colors using the weights (probs)
    # C = sum(w_i * c_i)
    final_color = np.sum(probs * colors)
    
    return final_color
```

### Edge Cases and Pitfalls
*   **Uneven Segment Lengths:** Using a sphere intersection representation creates uneven segment lengths, creating computational pressure on the network. TermiNeRF handles this by enforcing a constant length segment $\ell=4.0$ [S1].
*   **Arbitrary Training Signals:** Training both networks jointly from the beginning is inefficient because the sampling network receives arbitrary signals from an untrained color network. Pre-training the color network first is required [S1].
*   **Unsampled Regions:** If the sampling network predicts low weights for a region, those regions are never sampled, and the color network never learns the geometry there. To prevent this, 64 equidistant points are added during joint fine-tuning to ensure density acquisition in unsampled regions [S1].
*   **False Positives:** A false-positive sampling network misprediction (predicting high weight in free space) poses no issue for the color network's correctness, but wastes computation [S1].

## Operating characteristics
*   **Speed:** Rendering time is reduced to $\le 1$ second for an image, compared to $\sim 30$ seconds for standard NeRF on an 800x800 image [S1].
*   **Speedup:** TermiNeRF achieves a 14x speedup relative to standard NeRF [S1]. This is slower than KiloNeRF (600x) or PlenOctrees (3000x) but is end-to-end trainable and does not require depth maps or spatial data structures [S1].
*   **Quality:** Achieves similar or higher PSNR/LPIPS than NeRF with significantly fewer forward passes (e.g., 7.32x speedup for TermiNeRF 32 vs NeRF 8+16) [S1].
*   **Fine-tuning:** Scenes can be fine-tuned up to the penultimate step in only three minutes on a GTX1080 [S1].
*   **Memory:** The sampling network has the same architecture size as a NeRF network (8 hidden layers, 256 units), adding minimal memory overhead [S1].
*   **Limitations:** Requires a pre-trained NeRF to initialize the color network for best results [S2].

## Background
TermiNeRF was introduced by Martin Piala and Ronald Clark from Imperial College London in 2021 [S1]. It addresses the slow rendering times of neural radiance fields by learning a direct mapping from camera rays to locations along the ray that are most likely to influence the pixel's final appearance [S1]. The method splits the NeRF network into a sampling network and a shading network, trained jointly [S2].

## Sources
[S1] TermiNeRF: Ray Termination Prediction for Efficient Neural Rendering, https://doi.org/10.1109/3dv53792.2021.00118, fetched 2026-10-26
[S2] Recent Advances and Trends in Learning‐Based 3D Representations, https://doi.org/10.1111/cgf.70530, fetched 2026-10-26
