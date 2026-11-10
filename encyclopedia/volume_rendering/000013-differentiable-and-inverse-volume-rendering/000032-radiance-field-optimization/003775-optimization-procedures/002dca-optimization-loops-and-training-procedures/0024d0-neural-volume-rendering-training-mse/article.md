# Neural Volume Rendering Training (MSE)
Also called: Hierarchical Geometric Guidance (HGG)

## Overview
This method defines the core optimization loop for training Neural Radiance Fields (NeRF) and similar implicit neural representations by minimizing the Mean Squared Error (MSE) between rendered images and ground truth observations. It addresses the problem of learning accurate density and color distributions from sparse or dense 2D image inputs without explicit 3D supervision. It sets itself apart from sibling methods like [NeRF Training Loop](970-nerf-training-loop.md) by explicitly incorporating Hierarchical Geometric Guidance (HGG) to initialize sampling regions based on sparse depth priors, thereby avoiding the geometric misalignment bias introduced by direct depth supervision [S1, unofficial].

## How it works
**Inputs and Outputs:**
- **Inputs:** A set of $M$ calibrated images with known camera poses, intrinsics, and potentially sparse depth priors from Structure-from-Motion (SfM) [S1, unofficial]. Ground truth RGB values $C_{gt}(r)$ for rays $r$ passing through each image pixel.
- **Outputs:** Optimized neural network parameters $\Theta$ (defining density $\sigma$ and color $\mathbf{c}$) and optimized near/far plane bounds for sampling.

**Data Structures:**
- **Neural Network:** A multi-layer perceptron (MLP) $f_\Theta$ mapping 3D coordinates $\mathbf{x}$ and viewing direction $\mathbf{d}$ to density $\sigma$ and color $\mathbf{c}$ [S1, unofficial].
- **Sampling Points:** A set of 3D points $\{t_i\}$ along each ray $\mathbf{r}$, bounded by near plane $t_n$ and far plane $t_f$.
- **Depth Prior:** Sparse depth values $t_{depth}$ associated with camera rays, derived from SfM [S1, unofficial].

**Numbered Steps:**

1.  **Initialization with Geometric Guidance:**
    Initialize the near plane $t_n$ and far plane $t_f$ for each ray close to the sparse depth prior $t_{depth}$ to define a local sampling region. This avoids the bias $\Delta P$ caused by keypoint mismatching and small translations between frames that occurs with direct depth supervision [S1, unofficial].
    
    The initial bounds are set such that the sampling region is centered around $t_{depth}$.

2.  **Hierarchical Region Expansion:**
    Gradually extend $t_n$ and $t_f$ from the local region to the full scene bounds over $N_{hgg}$ iterations using a cosine annealing schedule $\gamma(i)$ [S1, unofficial].
    
    The bounds at iteration $i$ are calculated as:
    $$
    t_n(i) = t_{depth} + (t_n - t_{depth}) \gamma(i)
    $$
    $$
    t_f(i) = t_{depth} + (t_f - t_{depth}) \gamma(i)
    $$
    where $t_n, t_f$ are the initial near/far planes, $t_{depth}$ is the depth prior, and $\gamma(i)$ is the region adjustment rate [S1, unofficial].
    
    The adjustment rate $\gamma(i)$ is defined as:
    $$
    \gamma(i) = \frac{1 - \cos\left( \left( \min\left( \max\left( \frac{i}{N_{hgg}}, \epsilon_{hgg} \right), 1 \right) \right) \pi \right)}{2}
    $$
    where $i$ is the current iteration, $N_{hgg}$ is the total number of iterations until full bounds are reached, and $\epsilon_{hgg}$ is the minimum adjusting rate [S1, unofficial].

3.  **Volume Rendering:**
    For each ray $\mathbf{r}$, sample $N$ points $t_i$ within the current bounds $[t_n(i), t_f(i)]$. Query the neural network $f_\Theta$ to obtain density $\sigma_i$ and color $\mathbf{c}_i$ at each point. Compute the transmittance $T_i$ and the rendered color $\hat{C}(\mathbf{r})$ using the standard emission-absorption model [44-radiance-field-optimization.md]:
    $$
    \hat{C}(\mathbf{r}) = \sum_{i=1}^N T_i (1 - \exp(-\sigma_i \delta_i)) \mathbf{c}_i
    $$
    where $\delta_i = t_{i+1} - t_i$ and $T_i = \exp\left(-\sum_{j=1}^{i-1} \sigma_j \delta_j\right)$ [S1, unofficial].

4.  **Loss Computation:**
    Compute the Mean Squared Error (MSE) loss between the rendered color $\hat{C}(\mathbf{r})$ and the ground truth color $C_{gt}(\mathbf{r})$ [S1, unofficial].
    $$
    L_{mse} = \frac{1}{|R|} \sum_{r \in R} \left( C_{gt}(r) - \hat{C}(r) \right)^2
    $$
    where $R$ is the set of all rays sampled from the training images [S1, unofficial].

5.  **Optimization Step:**
    Update the network parameters $\Theta$ using an optimizer (e.g., Adam) to minimize $L_{mse}$ [S1, unofficial].

**Parameters:**
- $N_{hgg}$: Number of iterations until full bounds are reached. Typical value: 10% of total training iterations [S1, unofficial].
- $\epsilon_{hgg}$: Minimum adjusting rate. Typical value: 0.2 [S1, unofficial].
- Learning Rate: Typically $1 \times 10^{-3}$, exponentially decaying to $1 \times 10^{-5}$ [S1, unofficial].
- Samples per Ray: 64 for coarse models, 128 for fine models [S1, unofficial].

**Edge Cases and Pitfalls:**
- **Depth Bias:** Direct supervision of sparse depth priors introduces geometric misalignment due to keypoint mismatching ($\Delta d$) and small translations between frames, causing bias $\Delta P$ [S1, unofficial]. HGG mitigates this by using depth only for sampling region initialization, not as a direct loss term.
- **Precision:** The exponential decay in transmittance calculation can suffer from numerical underflow if $\sigma_i \delta_i$ is large; standard implementations use $\exp(-\sigma_i \delta_i)$ carefully or clamp values [G].
- **Boundary Conditions:** If $t_{depth}$ is missing for a ray, fallback to standard global bounds $[t_n, t_f]$ [G].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import torch
import torch.nn as nn
import torch.optim as optim
import math

class NeuralVolumeRenderingMSE:
    def __init__(self, nerf_model, depth_priors, image_height, image_width, 
                 n_hgg, epsilon_hgg, total_iterations, lr=1e-3):
        """
        Initialize the training loop.
        
        Args:
            nerf_model: Neural network f_theta predicting sigma and color.
            depth_priors: Dictionary or tensor of sparse depth values t_depth for rays.
            image_height, image_width: Dimensions of input images.
            n_hgg: Number of iterations for hierarchical geometric guidance expansion.
            epsilon_hgg: Minimum adjusting rate for gamma.
            total_iterations: Total number of training iterations.
            lr: Initial learning rate.
        """
        self.model = nerf_model
        self.depth_priors = depth_priors
        self.H = image_height
        self.W = image_width
        self.n_hgg = n_hgg
        self.epsilon_hgg = epsilon_hgg
        self.total_iterations = total_iterations
        self.optimizer = optim.Adam(nerf_model.parameters(), lr=lr)
        
        # Initial bounds (can be fixed or derived from scene bounds)
        # For demonstration, assuming global bounds are known or fixed
        self.t_n_global = 0.0 
        self.t_f_global = 10.0 
        
    def get_gamma(self, i):
        """Calculate the region adjustment rate gamma(i)."""
        ratio = i / self.n_hgg
        clamped_ratio = max(min(ratio, 1.0), self.epsilon_hgg)
        return (1 - math.cos(clamped_ratio * math.pi)) / 2.0

    def get_bounds(self, t_depth, i):
        """
        Calculate current near and far planes based on HGG.
        
        Args:
            t_depth: Sparse depth prior for the ray.
            i: Current iteration.
            
        Returns:
            t_n, t_f: Current near and far planes.
        """
        gamma = self.get_gamma(i)
        # If no depth prior, use global bounds (or a default local region)
        if t_depth is None:
            return self.t_n_global, self.t_f_global
            
        t_n = t_depth + (self.t_n_global - t_depth) * gamma
        t_f = t_depth + (self.t_f_global - t_depth) * gamma
        return t_n, t_f

    def render_ray(self, ray_origin, ray_direction, t_n, t_f, n_samples=64):
        """
        Render a single ray using the volume rendering equation.
        
        Args:
            ray_origin: 3D vector, camera origin.
            ray_direction: 3D vector, ray direction.
            t_n: Near plane distance.
            t_f: Far plane distance.
            n_samples: Number of samples along the ray.
            
        Returns:
            color: Rendered RGB color.
        """
        # Sample points uniformly between t_n and t_f
        t_vals = torch.linspace(t_n, t_f, n_samples)
        # Add jitter for stratified sampling (optional but common)
        # t_vals = t_vals + torch.rand_like(t_vals) * (t_f - t_n) / n_samples
        
        # Compute deltas
        deltas = t_vals[1:] - t_vals[:-1]
        delta_last = t_f - t_vals[-1]
        deltas = torch.cat([deltas, delta_last.unsqueeze(0)])
        
        # Sample points in 3D
        points_3d = ray_origin.unsqueeze(0) + ray_direction.unsqueeze(0) * t_vals.unsqueeze(1)
        
        # Query NeRF for sigma and color
        # Assuming model returns (sigma, color)
        sigma, color = self.model(points_3d)
        
        # Compute transmittance
        # alpha = 1 - exp(-sigma * delta)
        alpha = 1.0 - torch.exp(-sigma * deltas)
        
        # Transmittance T_i = exp(-sum_{j=1}^{i-1} sigma_j * delta_j)
        # Cumulative product of (1 - alpha)
        transmittance = torch.cumprod(torch.cat([torch.ones_like(alpha[:1]), 1.0 - alpha[:-1]]), dim=0)
        
        # Rendered color
        # C = sum_{i=1}^N T_i * alpha_i * c_i
        weights = transmittance * alpha
        rendered_color = torch.sum(weights.unsqueeze(-1) * color, dim=0)
        
        return rendered_color

    def train_step(self, ray_origins, ray_directions, gt_colors, iteration):
        """
        Perform one training step.
        
        Args:
            ray_origins: Tensor of shape (H*W, 3) or (batch_size, H*W, 3).
            ray_directions: Tensor of shape (H*W, 3) or (batch_size, H*W, 3).
            gt_colors: Tensor of shape (H*W, 3) or (batch_size, H*W, 3).
            iteration: Current global iteration index.
            
        Returns:
            loss: Scalar MSE loss.
        """
        self.optimizer.zero_grad()
        
        # Determine batch size or process all rays
        # For simplicity, assume all rays are processed in one forward pass
        # In practice, this is often batched
        
        n_rays = ray_origins.shape[0]
        rendered_colors = torch.zeros_like(gt_colors)
        
        # Process rays in chunks to manage memory if necessary
        chunk_size = 1024
        for i in range(0, n_rays, chunk_size):
            chunk_origins = ray_origins[i:i+chunk_size]
            chunk_directions = ray_directions[i:i+chunk_size]
            
            # Get depth priors for this chunk
            # Assuming depth_priors is indexed by ray index or matched by position
            # This is a simplification; in practice, depth priors are matched to rays
            chunk_depths = self.get_depth_priors_for_rays(i, chunk_size)
            
            # Calculate bounds for this chunk
            # Note: t_n_global and t_f_global are assumed to be scalars or broadcastable
            # In HGG, t_n and t_f are initialized close to t_depth.
            # Here we assume t_n_global and t_f_global are the "full bounds" 
            # and t_depth is the "local" center.
            # The formula in S1: t_n(i) = t_depth + (t_n - t_depth) * gamma(i)
            # where t_n, t_f are initial near/far planes. 
            # Wait, S1 says "initialize ... close to depth prior ... then gradually extend ... to full scene bounds".
            # So t_n_global and t_f_global in my init are the "full scene bounds".
            # The "initial near/far planes" in the formula are effectively t_depth +/- small_offset?
            # Actually, S1 formula: t_n(i) = t_depth + (t_n - t_depth) * gamma(i).
            # If gamma(0) = epsilon, and gamma(N_hgg) = 1.
            # At i=0, t_n = t_depth + (t_n - t_depth)*epsilon.
            # At i=N_hgg, t_n = t_depth + (t_n - t_depth)*1 = t_n.
            # This implies t_n and t_f in the formula are the FINAL bounds (full scene bounds).
            # And the "initial" bounds are close to t_depth.
            # Let's re-read carefully: "initialize ... setting near planes tn and tf close to depth prior ... 
            # Then tn and tf are gradually extended to the full scene bounds".
            # Formula: tn(i) = t_depth + (tn - t_depth) * gamma(i).
            # If tn is the full scene bound, then at i=N_hgg, tn(i) = tn. Correct.
            # At i=0, tn(0) = t_depth + (tn - t_depth)*epsilon. This is close to t_depth if epsilon is small.
            # So t_n_global and t_f_global in my code are the "tn" and "tf" in the formula.
            
            t_n_vals, t_f_vals = [], []
            for d in chunk_depths:
                tn, tf = self.get_bounds(d, iteration)
                t_n_vals.append(tn)
                t_f_vals.append(tf)
            
            t_n_tensor = torch.tensor(t_n_vals, device=ray_origins.device).unsqueeze(1)
            t_f_tensor = torch.tensor(t_f_vals, device=ray_origins.device).unsqueeze(1)
            
            # Render chunk
            # Broadcasting: origins (N, 3), t_n (N, 1)
            # We need to call render_ray for each ray or vectorize it.
            # Vectorized render_ray is complex. For pseudocode, we assume a batched version or loop.
            # Let's assume a batched render function exists or loop.
            
            for j in range(chunk_origins.shape[0]):
                rendered_colors[i+j] = self.render_ray(
                    chunk_origins[j], 
                    chunk_directions[j], 
                    t_n_tensor[j], 
                    t_f_tensor[j]
                )
        
        # Compute Loss
        loss = torch.mean((rendered_colors - gt_colors) ** 2)
        
        # Backward pass
        loss.backward()
        self.optimizer.step()
        
        return loss.item()

    def get_depth_priors_for_rays(self, start_idx, count):
        """
        Helper to retrieve depth priors for a batch of rays.
        Implementation depends on how depth priors are stored.
        """
        # Placeholder for actual retrieval logic
        # Returns a list of depth values or None
        return [None] * count
```

## Subtopics

### [Neural Surface Rendering Training (Eikonal & Boundary Regularization)](3430-neural-surface-rendering-training-eikonal-boundary-regulariz.md)
Trains a neural Signed Distance Function (SDF) to render images matching ground truth RGB while enforcing geometric priors via Eikonal and boundary regularization losses. It differs from standard neural volume rendering training (MSE) by explicitly optimizing the gradient magnitude of the SDF to be unity (Eikonal equation) and penalizing non-zero SDF values in the volume (boundary loss), which stabilizes surface extraction and improves geometric accuracy.

### [Global Optimization](5436-global-optimization.md)
Global Optimization is a method for reconstructing a complete image or volume from a sparse subset of measurements by solving for all pixels/voxels simultaneously across the entire domain. It sets itself apart from local or block-based solvers by considering the global context of the entire image during the optimization process, which can improve reconstruction quality at the cost of higher memory usage and computational complexity per iteration. It is primarily used in compressive rendering and tomographic reconstruction where data is incomplete.

### [Neural Volume Rendering Training Setup](6559-neural-volume-rendering-training-setup.md)
This article details the specific hyperparameters, data preprocessing, and hardware configurations required to train neural volume rendering models, focusing on the implicit neural representation (INR) approach for medical volume data. It establishes the baseline training loop for reconstructing high-resolution volumes from downsampled inputs using an L1 loss and Adam optimizer. A minor variant, Mobile-GS, adapts this setup for 3D Gaussian Splatting with distillation and quantization.

### [Handpainted Reference Image Inversion](6686-handpainted-reference-image-inversion.md)
Handpainted Reference Image Inversion is a training task where a volume rendering model learns to reproduce a user-edited 2D reference image from a 3D volume, demonstrating the ability to learn complex classification features that are difficult to define with traditional multidimensional transfer functions. It solves the problem of tedious trial-and-error transfer function adjustments by allowing the model to automatically learn the parameters required to replicate a user-provided visual effect. Unlike base 3D Gaussian Splatting which bakes the scene appearance, this method enables editing of lighting and appearance via optimization, achieving perceptual fidelity even from poor viewpoint reference images. It sets itself apart from sibling methods like Global Optimization by focusing on semantic colorization tasks derived from hand-painted edits rather than pure photometric reconstruction or geometric regularization.

### [Coarse-to-fine Sparse Volume Training](6838-coarse-to-fine-sparse-volume-training.md)
This method defines the training and optimization pipeline for sparse neural radiance fields, specifically focusing on the fine-tuning stage that refines a pre-constructed sparse volume $V^g$ per scene. It solves the problem of achieving high-fidelity novel view synthesis with significantly reduced training time compared to dense per-scene optimization methods like NeRF. It is distinguished from its siblings by its reliance on a coarse-to-fine iterative pruning and subdivision strategy during optimization, rather than static initialization or global optimization over the entire dataset. It builds on [Neural Volume Rendering Training (MSE)](3429-neural-volume-rendering-training-mse.md).

### [INR Training Loop](10522-inr-training-loop.md)
The Implicit Neural Representation (INR) training loop optimizes a neural network to approximate a discrete volumetric dataset by minimizing the mean squared error between network predictions and ground-truth values sampled from the volume. It differs from explicit volume rendering methods by learning a continuous function $f_\theta$ rather than storing discrete voxels, enabling resolution-independent querying.

### Inverse Rendering for Texture/Attribute Refinement
Refinement: Use inverse rendering to optimize TexF attributes. Propagate image-space supervision through a differentiable rendering process. Compute a photometric loss between UV reference images and TexF renderings. Use this loss to guide voxel attribute refinement [S2, unofficial]. Optimization Target: The refinement reduces transfer and discretization errors during representation construction and refines the compressed neural field [S2, unofficial]. The optimization minimizes a photometric loss between reference images (from UV rendering) and rendered images (from TexF rendering) [S2, unofficial].

### Hierarchical Photometric Guidance
Trains a coarse model and a fine model to improve novel view synthesis performance by combining their predicted colors in the loss function [S1, unofficial]. Steps: 1. Query coarse model $f_\Theta$ with stratified volume samples $t_c$ to get $\hat{C}_{crecon}$. 2. Query fine model with union of coarse and fine samples $t_c \cup t_f$ (produced by inverse transform sampling) to get $\hat{C}_{frecon}$. 3. Compute MSE loss against ground truth $C_{gt}$ for both predictions [S1, unofficial]. Formula: $L_{hpg} = \frac{1}{2} \sum_{r \in R} \left( C_{gt}(r) - \hat{C}_{crecon}(r; t_c) \right)^2 + \left( C_{gt}(r) - \hat{C}_{frecon}(r; t_c \cup t_f) \right)^2$, where $R$ is the set of all rays, $t_c$ are coarse stratified samples, and $t_f$ are fine samples produced by inverse transform sampling [S1, unofficial].

## Operating characteristics
- **Time Complexity:** $O(N \cdot M \cdot S)$, where $N$ is the number of rays, $M$ is the number of network layers/parameters, and $S$ is the number of samples per ray. The HGG expansion adds a negligible overhead to bound calculation.
- **Memory Complexity:** $O(N \cdot S)$ for storing intermediate rendering states during backpropagation if using reverse-mode AD. Adjoint methods can reduce this to $O(N)$ [44-radiance-field-optimization.md].
- **Measured Performance:** On the DTU dataset, HG3-NeRF (using HGG) synthesizes images with more fine details and estimates depth maps with distinct edges compared to SOTA baselines [S1, unofficial].
- **Scaling:** Scales with the number of training views and resolution. HGG allows effective training with sparse views (3, 6, 9 views) by guiding the initial density distribution [S1, unofficial].
- **Strengths:** Avoids geometric bias from sparse depth priors; improves novel view synthesis quality; simple loss function (MSE).
- **Limitations:** Relies on the quality of sparse depth priors for initialization; may converge to local minima if bounds are poorly initialized; standard NeRF training can be slow due to volume rendering integration.

## Background
The method originates from the work on Hierarchical Geometric, Semantic, and Photometric Guided Neural Radiance Fields (HG3-NeRF) [S1, unofficial]. It addresses the limitations of standard NeRF training when dealing with sparse input views and sparse depth priors, where direct depth supervision can introduce geometric misalignment. The HGG technique was introduced to leverage sparse depth information for sampling region initialization without propagating its errors into the final scene representation.

## Sources
[S1] HG3-NeRF: Hierarchical Geometric, Semantic, and Photometric Guided Neural Radiance Fields for Sparse View Inputs, https://doi.org/10.48550/arXiv.2401.11711, fetched 2026-10-29 (unofficial)
[S2] Beyond UV Mapping: Mesh Texture Compression via Surface-Aligned Texture Fields, https://doi.org/10.48550/arxiv.2609.23606, fetched 2026-10-26 (unofficial)
