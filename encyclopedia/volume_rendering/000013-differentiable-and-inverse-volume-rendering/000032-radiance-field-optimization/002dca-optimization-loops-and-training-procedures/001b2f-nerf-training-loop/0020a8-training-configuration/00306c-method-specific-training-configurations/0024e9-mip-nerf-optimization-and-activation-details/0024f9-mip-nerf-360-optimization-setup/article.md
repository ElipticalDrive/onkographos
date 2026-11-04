# Mip-NeRF 360 Optimization Setup

Also called: Mip-NeRF 360 Training Procedure

## Overview
This article specifies the hyperparameters, network architectures, and loss functions used to train the Mip-NeRF 360 model [S1]. It stabilizes optimization by combining a two-stage proposal sampling strategy with a Charbonnier reconstruction loss and a stop-gradient operation that decouples proposal network updates from NeRF updates [S1]. It differs from standard NeRF training by enforcing strict separation between proposal and NeRF optimization via stop-gradient operations, and by using a Charbonnier loss instead of mean squared error to achieve more stable optimization [S1].

## How it works

### Inputs and Outputs
*   **Inputs**: A batch of rays $\mathbf{t}$, each defined by origin and direction, and their corresponding ground-truth pixel colors $\mathbf{C}^*$ [S1].
*   **Outputs**: Optimized weights for the proposal MLP ($\Theta_{\text{prop}}$) and the NeRF MLP ($\Theta_{\text{NeRF}}$) [S1].

### Data Structures
The method relies on two Multi-Layer Perceptrons (MLPs):
1.  **Proposal MLP**:
    *   Architecture: 4 layers, 256 hidden units per layer [S1].
    *   Activations: ReLU for hidden layers, Softplus for the final density output $\tau$ [S1].
    *   Parameters: $\Theta_{\text{prop}}$ [S1].
2.  **NeRF MLP**:
    *   Architecture: 8 layers, 1024 hidden units per layer [S1].
    *   Activations: ReLU for hidden layers, Softplus for the final density output $\tau$ [S1].
    *   Parameters: $\Theta_{\text{NeRF}}$ [S1].

### Numbered Steps
1.  **Proposal Sampling Stage 0**: Evaluate the proposal MLP using 64 samples along each ray to produce the first set of weighted intervals $(\hat{s}_0, \hat{w}_0)$ [S1].
2.  **Proposal Sampling Stage 1**: Resample and evaluate the proposal MLP again using 64 samples to produce the second set of weighted intervals $(\hat{s}_1, \hat{w}_1)$ [S1].
3.  **NeRF Sampling**: Evaluate the NeRF MLP using 32 samples, resampled based on the proposal histograms, to produce the final set of weighted intervals $(s, w)$ [S1].
4.  **Loss Computation**: Compute the total loss $\mathcal{L}$ as the sum of reconstruction, distribution, and proposal losses, averaged over all rays in the batch [S1]:
    $$ \mathcal{L} = \mathcal{L}_{\text{recon}}(\mathbf{C}(\mathbf{t}), \mathbf{C}^*) + \lambda \mathcal{L}_{\text{dist}}(\mathbf{s}, \mathbf{w}) + \sum_{k=0}^1 \mathcal{L}_{\text{prop}}(\mathbf{s}, \mathbf{w}, \hat{\mathbf{s}}^k, \hat{\mathbf{w}}^k) $$
5.  **Gradient Application**: Apply stop-gradient to $\mathcal{L}_{\text{prop}}$ before backpropagating to $\Theta_{\text{NeRF}}$ to make the optimization of $\Theta_{\text{prop}}$ independent from $\Theta_{\text{NeRF}}$ [S1].
6.  **Optimizer Update**: Update both networks using the Adam optimizer [S1].

### Formulas
*   **Reconstruction Loss** ($\mathcal{L}_{\text{recon}}$): Charbonnier loss [S1]:
    $$ \mathcal{L}_{\text{recon}} = \sqrt{(x - x^*)^2 + \epsilon^2} $$
    Where $x$ is the predicted color, $x^*$ is the ground truth color, and $\epsilon = 0.001$ [S1].
*   **Distribution Loss** ($\mathcal{L}_{\text{dist}}$): Minimizes the weighted size of each individual interval [S1].
*   **Regularization Weight** ($\lambda$): Balances the data term $\mathcal{L}_{\text{recon}}$ and the regularizer $\mathcal{L}_{\text{dist}}$, set to $0.01$ in all experiments [S1].
*   **Total Loss** ($\mathcal{L}$): As defined in Step 4 above [S1].

### Parameters
*   **Optimizer**: Adam [S1].
    *   $\beta_1 = 0.9$
    *   $\beta_2 = 0.999$
    *   $\epsilon = 10^{-6}$
*   **Learning Rate Schedule**: Annealed log-linearly from $2 \times 10^{-3}$ to $2 \times 10^{-5}$ [S1].
    *   Warm-up phase: 512 iterations [S1].
*   **Gradient Clipping**: Norm of $10^{-3}$ [S1].
*   **Training Iterations**: 250,000 [S1].
*   **Batch Size**: $2^{14}$ rays [S1].
*   **Regularization Weight** ($\lambda$): 0.01 [S1].

### Edge Cases and Pitfalls
*   **Z-aliasing**: The proposal MLP may learn a non-smooth mapping from input coordinates to volumetric densities, causing rays to "skip" scene content [S2]. This artifact is particularly visible when the camera translates along the z-axis or when using high-capacity backends like iNGP [S2].
*   **Interlevel Loss Artifacts**: The standard Mip-NeRF 360 interlevel loss assigns equivalent loss to NeRF and proposal histogram bins regardless of overlap extent, penalizing only complete failures to overlap, which leads to non-smooth density maps [S2].

### Pseudocode
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def train_mipnerf360(batch_rays, batch_gt_colors, num_iterations=250000):
    # Initialize networks
    proposal_mlp = MLP(layers=4, hidden_units=256, activation='relu', density_activation='softplus')
    nerf_mlp = MLP(layers=8, hidden_units=1024, activation='relu', density_activation='softplus')
    
    # Initialize optimizer
    optimizer = Adam(params=[proposal_mlp.parameters(), nerf_mlp.parameters()], 
                     lr=2e-3, beta1=0.9, beta2=0.999, eps=1e-6)
    
    # Hyperparameters
    lr_start = 2e-3
    lr_end = 2e-5
    warmup_iters = 512
    clip_norm = 1e-3
    lambda_reg = 0.01
    epsilon_charb = 0.001
    
    for iteration in range(1, num_iterations + 1):
        # 1. Proposal Sampling Stage 0
        t_0 = uniform_sample_t(num_samples=64)
        densities_0 = proposal_mlp(batch_rays, t_0)
        s_hat_0, w_hat_0 = construct_histogram(t_0, densities_0)
        
        # 2. Proposal Sampling Stage 1
        t_1 = importance_sample_t(histogram=(s_hat_0, w_hat_0), num_samples=64)
        densities_1 = proposal_mlp(batch_rays, t_1)
        s_hat_1, w_hat_1 = construct_histogram(t_1, densities_1)
        
        # 3. NeRF Sampling
        t_nerf = importance_sample_t(histogram=(s_hat_1, w_hat_1), num_samples=32)
        densities_nerf, colors_nerf = nerf_mlp(batch_rays, t_nerf)
        s, w = construct_histogram(t_nerf, densities_nerf)
        
        # 4. Compute Losses
        rendered_colors = render_volume(s, w, colors_nerf)
        recon_loss = charbonnier_loss(rendered_colors, batch_gt_colors, epsilon_charb)
        dist_loss = interval_size_loss(s, w)
        prop_loss_0 = interlevel_loss(s, w, s_hat_0, w_hat_0)
        prop_loss_1 = interlevel_loss(s, w, s_hat_1, w_hat_1)
        total_prop_loss = prop_loss_0 + prop_loss_1
        
        # Total loss formula
        total_loss = recon_loss + lambda_reg * dist_loss + total_prop_loss
        
        # 5. Gradient Application with stop-gradient on L_prop for NeRF
        # Compute gradients for proposal network (includes L_prop)
        grads_prop = compute_gradients(total_loss, proposal_mlp.parameters())
        # Compute gradients for NeRF network (excludes L_prop via stop-gradient)
        loss_nerf_only = recon_loss + lambda_reg * dist_loss
        grads_nerf = compute_gradients(loss_nerf_only, nerf_mlp.parameters())
        
        # 6. Optimizer Update
        clip_gradients(grads_prop, clip_norm)
        clip_gradients(grads_nerf, clip_norm)
        optimizer.step(grads_prop, grads_nerf)
        
        # Update learning rate
        if iteration < warmup_iters:
            current_lr = lr_start * (iteration / warmup_iters)
        else:
            progress = (iteration - warmup_iters) / (num_iterations - warmup_iters)
            current_lr = lr_start * (lr_end / lr_start) ** progress
        optimizer.set_lr(current_lr)

# Helper definitions
def MLP(layers, hidden_units, activation, density_activation):
    """Standard feedforward network with specified activations."""
    return Network(layers, hidden_units, activation, density_activation)

def Adam(params, lr, beta1, beta2, eps):
    """Standard Adam optimizer with momentum and RMSprop scaling."""
    return Optimizer(params, lr, beta1, beta2, eps)

def uniform_sample_t(num_samples):
    """Draws uniform samples from [0, 1]."""
    return random.uniform(0, 1, size=num_samples)

def importance_sample_t(histogram, num_samples):
    """Draws samples proportional to histogram weights."""
    s, w = histogram
    cdf = cumulative_sum(w)
    return inverse_cdf_sample(cdf, num_samples)

def construct_histogram(t, densities):
    """Converts raw samples to piecewise-linear histogram midpoints (s) and weights (w)."""
    s = (t[:-1] + t[1:]) / 2
    w = densities * (t[1:] - t[:-1])
    return s, w

def render_volume(s, w, colors):
    """Renders color from histogram using standard volume rendering accumulation."""
    return accumulate_transmittance_and_color(s, w, colors)

def charbonnier_loss(pred, target, epsilon):
    """Computes Charbonnier loss."""
    return sqrt((pred - target)**2 + epsilon**2)

def interval_size_loss(s, w):
    """Minimizes the weighted size of each individual interval."""
    return mean(w * (s[1:] - s[:-1]))

def interlevel_loss(s_nerf, w_nerf, s_prop, w_prop):
    """Computes distance between NeRF and proposal histograms."""
    return histogram_distance(s_nerf, w_nerf, s_prop, w_prop)

def compute_gradients(loss, params):
    """Computes gradients of loss with respect to parameters."""
    return autograd.grad(loss, params)

def clip_gradients(grads, max_norm):
    """Clips gradient norms to max_norm."""
    norm = sqrt(sum(g**2 for g in grads))
    if norm > max_norm:
        scale = max_norm / norm
        return [g * scale for g in grads]
    return grads
```

## Subtopics

### [Anti-Aliased Interlevel Loss](3457-anti-aliased-interlevel-loss.md)
The Anti-Aliased Interlevel Loss is a proposal supervision loss function that measures the distance between a NeRF histogram and a proposal histogram by blurring the NeRF histogram into a piecewise-linear spline and resampling it into the proposal's interval endpoints. Unlike the piecewise constant loss used in Mip-NeRF 360, which suffers from aliasing when NeRF histogram bins are narrow or translate relative to the proposal, this method produces a smooth loss surface that prevents aliasing artifacts during training.

### Smooth Proposal Supervision
This method resolves z-aliasing artifacts in proposal networks by replacing the discrete interlevel loss with a continuous, smooth loss function that penalizes partial overlaps between proposal and NeRF histograms [S2]. It addresses the pitfall where the original Mip-NeRF 360 interlevel loss assigns equivalent loss to NeRF and proposal histogram bins regardless of overlap extent, penalizing only complete failures to overlap and leading to non-smooth density maps [S2]. The new loss prevents the proposal network from learning non-smooth mappings that occur with the previous loss, particularly when the proposal network has high capacity such as an iNGP backend [S2].

## Operating characteristics

*   **Time Complexity**: The method uses a multi-stage sampling approach (2 proposal stages + 1 NeRF stage), increasing the number of forward passes per ray compared to single-stage NeRF [S1]. The proposal network is smaller (4 layers, 256 units) than the NeRF (8 layers, 1024 units), mitigating the computational cost [S1].
*   **Memory Complexity**: Memory usage is dominated by the storage of the two MLPs and the intermediate histograms for the batch [G]. The batch size of $2^{14}$ rays requires significant memory for ray data and intermediate tensors [S1].
*   **Measured Performance**: The method was evaluated on 9 scenes (5 outdoors and 4 indoors) each containing a complex central object or area and a detailed background [S1]. It improves speed and rendering quality compared to prior strategies of learning one or multiple NeRFs that are all supervised using image reconstruction [S1].
*   **Strengths**: The use of Charbonnier loss achieves slightly more stable optimization than the mean squared error used in mip-NeRF [S1]. The stop-gradient on proposal loss ensures independent optimization of the proposal network [S1].
*   **Limitations**: The standard interlevel loss can lead to z-aliasing artifacts, particularly with high-capacity backends like iNGP [S2]. The method requires careful tuning of the learning rate schedule and gradient clipping [S1].
*   **When to Use**: Use this setup when training unbounded, anti-aliased radiance fields where optimization stability is critical [S1].
*   **When Not to Use**: Avoid if computational resources are extremely limited, as the multi-stage sampling increases forward pass count [S1]. Consider the Anti-Aliased Interlevel Loss variant if z-aliasing is a major concern and an iNGP backend is used [S2].

## Background

The Mip-NeRF 360 optimization setup was introduced in the paper "Mip-NeRF 360: Unbounded Anti-Aliased Neural Radiance Fields" [S1]. It builds on the original Mip-NeRF method and addresses challenges in training on unbounded datasets [S1]. The method was designed to improve training stability and robustness without significantly changing accuracy or speed [Parent]. The issue of z-aliasing in proposal networks was further analyzed in "Zip-NeRF: Anti-Aliased Grid-Based Neural Radiance Fields", which proposed the Anti-Aliased Interlevel Loss as a solution [S2].

## Sources

[S1] Mip-NeRF 360: Unbounded Anti-Aliased Neural Radiance Fields, https://doi.org/10.1109/cvpr52688.2022.00539, fetched 2026-10-26
[S2] Zip-NeRF: Anti-Aliased Grid-Based Neural Radiance Fields, https://doi.org/10.1109/iccv51070.2023.01804, fetched 2026-10-26
