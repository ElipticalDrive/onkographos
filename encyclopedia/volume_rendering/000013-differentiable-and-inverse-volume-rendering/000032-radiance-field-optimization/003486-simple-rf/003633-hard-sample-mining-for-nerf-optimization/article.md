# Hard Sample Mining for NeRF Optimization

Also called: Propagated Pixel Loss Importance Sampling, Two-Forward-One-Backward Hard Sample Mining, Two-forward-one-backward-pass approach

## Overview
Hard Sample Mining for NeRF Optimization is a training acceleration technique that reduces memory and compute costs by identifying and backpropagating gradients only for "hard" samples (those contributing significantly to the loss), while skipping the expensive backward pass for "easy" samples [S1]. It sets itself apart from sibling methods like [NeRF with Softplus Activation and Importance Sampling](3524-nerf-with-softplus-activation-and-importance-sampling.md) and [PlaNeRF](7394-planerf.md) by pruning samples based on their gradient contribution to the pixel loss rather than density or occupancy weights, and by using a two-forward-one-backward pass structure to bypass standard deep learning framework overheads associated with masked gradients [S1].

## How it works
**Inputs and Outputs:**
*   **Inputs:** A batch of rays sampled from training images, ground truth pixel colors, and the current state of the NeRF network (MLPs for density $\sigma$ and color $c$).
*   **Outputs:** Updated NeRF network parameters.

**Data Structures and Layout:**
*   **Points:** A tensor of shape $(B, 3)$ containing 3D coordinates of $B$ ray samples.
*   **Pre-activation Attributes ($Z$):** A tuple $(c', \sigma')$ of shape $(B, 3)$ and $(B, 1)$ respectively, representing the raw outputs of the color and density MLPs before activation functions (e.g., sigmoid, exp) [S1].
*   **Gradient Norms ($G$):** A tensor of shape $(B,)$ where $G_i = \|\frac{\partial L}{\partial (c'_i, \sigma'_i)}\|_2$ [S1].
*   **Hard Indices:** A 1D tensor of indices selecting $b$ samples from the original $B$ [S1].

**Formulas:**
1.  **Volumetric Rendering:** Pixel color $C(r)$ is computed as [S1]:
    $$C(r) = \sum_{i=1}^{N} w_i c_i, \quad w_i = T_i \alpha_i, \quad T_i = \prod_{j=1}^{i-1} (1 - \alpha_j), \quad \alpha_i = 1 - \exp(-\sigma_i \delta_i)$$
    where $c_i$ is color, $\sigma_i$ is density, $\delta_i$ is step size, $T_i$ is transmittance, and $\alpha_i$ is opacity.

2.  **Importance Sampling Distribution ($G$):** The hardness of a sample is defined by the L2 norm of the gradient of the pixel loss $L$ with respect to the pre-activation outputs [S1]:
    $$G_i = \left\| \frac{\partial L}{\partial (c'_i, \sigma'_i)} \right\|_2$$

3.  **Variance of Gradient Reduction Ratio ($R$):** Approximated using the large minibatch $B$ [S1]:
    $$R = \frac{1}{\sum_{i=1}^B G_i^2} \|G - U\|_2^2$$
    where $U = 1/B$ is the uniform distribution probability.

4.  **Batch Size Increase Factor ($\tau$):** Solved assuming gradient variance reduces to $1 - 1/\sqrt{\tau}$ when batch size increases by $\tau$ [S1]:
    $$\tau = \left(1 - R\right)^{-\frac{1}{2}} = \left(1 - \frac{1}{\sum_{i=1}^B G_i^2} \|G - U\|_2^2\right)^{-\frac{1}{2}}$$

5.  **Hard Minibatch Size ($b$):** Calculated to achieve the same gradient variance as the uniform batch $B$ [S1]:
    $$b = \frac{B}{\tau}$$

**Parameters:**
*   $B$: Total number of ray samples in the initial batch. Typical value: $2^{20}$ [S1].
*   $b$: Number of hard samples selected. Dynamically calculated; typically 10-15% of $B$ [S1].
*   $\hat{\tau}$: Running average of $\tau$, initialized to 1.0 [S1].
*   $\alpha_\tau$: Learning rate for $\hat{\tau}$, set to $1 / \text{num. of training images}$ [S1].

**Steps:**
1.  **Ray Sampling:** Cast rays through image pixels and generate a batch of $B$ ray samples (points along the rays) [S1].
2.  **First Forward Pass (Inference):** Perform the forward pass for all $B$ samples in inference mode (e.g., `no_grad()`) to avoid storing intermediate tensors for the backward pass [S1]. Retrieve pre-activation outputs $Z = (c', \sigma')$ [S1].
3.  **Volume Rendering and Loss:** Volume render the outputs from the first pass to calculate pixel colors, then compute the pixel loss $L$ [S1].
4.  **Partial Backpropagation:** Backpropagate the loss from the volume rendering stage only up to the pre-activation outputs $Z$ for all $B$ samples. This provides gradients $\frac{\partial L}{\partial Z}$ without backpropagating through the entire network architecture [S1].
5.  **Compute Importance Weights:** Calculate the norm of the gradient for each sample to form $G$ [S1].
6.  **Calculate Batch Size:** Compute $R$, then $\tau$, and update the running average $\hat{\tau} = (1 - \alpha_\tau)\hat{\tau} + \alpha_\tau \tau$. Calculate the target hard batch size $b = B / \hat{\tau}$ [S1].
7.  **Select Hard Samples:** Sample $b$ indices from the importance distribution $G$ using multinomial sampling: $hard\_indices \leftarrow \text{multinomial}(G, b)$ [S1]. Retrieve the corresponding 3D coordinates: $hard\_points \leftarrow points[hard\_indices]$ [S1].
8.  **Second Forward Pass (Training):** Re-run the forward pass for the selected $b$ hard samples, this time building the full computational graph (training mode) [S1]. Retrieve pre-activation attributes $attributes'$ [S1].
9.  **Full Backward Pass:** Concatenate the cached gradients for the selected hard indices: $grad\_output \leftarrow \text{cat}(grad\_attributes[hard\_indices])$. Backpropagate these cached gradients through the network for the hard points: $attributes'.\text{backward}(grad\_output)$ [S1].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def hard_sample_mining_step(nerf, points, gt_colors, alpha_tau, tau_hat):
    """
    Performs one iteration of hard sample mining.
    
    Args:
        nerf: The NeRF network model.
        points: Tensor of shape (B, 3) containing ray samples.
        gt_colors: Tensor of ground truth pixel colors.
        alpha_tau: Scalar learning rate for running average of tau.
        tau_hat: Scalar running average of tau.
        
    Returns:
        new_tau_hat: Updated running average of tau.
    """
    B = points.shape[0]
    
    # Step 1 & 2: First forward pass in inference mode
    with torch.no_grad():
        # Get pre-activation outputs (c', sigma')
        # Note: nerf must support returning pre-activation values
        c_prime, sigma_prime = nerf(points, preactivation=True)
        
        # Step 3: Volume render and compute loss
        # Assume volumerendering handles the accumulation of weights and colors
        # based on sigma_prime and c_prime (after activation)
        # We need activations for rendering, but gradients are w.r.t pre-activations
        alpha = 1 - torch.exp(-sigma_prime) # Opacity
        # Transmittance calculation would typically happen inside volumerendering
        # For simplicity, assuming volumerendering takes pre-activations or 
        # we reconstruct the rendering here. 
        # In practice, volumerendering takes sigma and c.
        # Let's assume we pass sigma_prime and c_prime to a renderer that 
        # handles the exponential and accumulation.
        rendered_colors = volumerendering(sigma_prime, c_prime)
        
        pixel_loss = loss_function(rendered_colors, gt_colors)
        
        # Step 4: Backpropagate only to pre-activation outputs
        # We need gradients of loss w.r.t c_prime and sigma_prime
        grad_c_prime, grad_sigma_prime = torch.autograd.grad(
            pixel_loss, 
            (c_prime, sigma_prime), 
            retain_graph=True, 
            create_graph=False
        )
        
        # Step 5: Compute importance weights G
        # G is the L2 norm of the gradient vector [grad_c_prime, grad_sigma_prime]
        # Assuming c_prime is (B, 3) and sigma_prime is (B, 1)
        grad_norms = torch.sqrt(
            torch.sum(grad_c_prime**2, dim=1) + 
            torch.sum(grad_sigma_prime**2, dim=1)
        )
        G = grad_norms
    
    # Step 6: Calculate batch size b
    U = 1.0 / B
    sum_G_sq = torch.sum(G**2)
    R = (1.0 / sum_G_sq) * torch.sum((G - U)**2)
    
    # Avoid division by zero or invalid sqrt if R >= 1
    if R >= 1.0:
        tau = 1.0
    else:
        tau = (1.0 - R)**(-0.5)
        
    new_tau_hat = (1.0 - alpha_tau) * tau_hat + alpha_tau * tau
    b = int(B / new_tau_hat)
    
    # Ensure b is at least 1
    if b < 1:
        b = 1
        
    # Step 7: Select hard samples
    # Multinomial sampling from G (probabilities)
    # G must be normalized to sum to 1 for multinomial
    probs = G / torch.sum(G)
    hard_indices = torch.multinomial(probs, b, replacement=False)
    hard_points = points[hard_indices]
    
    # Step 8: Second forward pass (Training mode)
    # Build computational graph
    c_prime_hard, sigma_prime_hard = nerf(hard_points, preactivation=True)
    
    # Step 9: Full backward pass
    # Retrieve cached gradients for the selected indices
    # Note: grad_c_prime and grad_sigma_prime are from the first pass (no_grad context)
    # We need to detach them or use the values computed in step 4
    # In the source algorithm, grad_attributes are cached.
    # Here we assume we have access to the gradients computed in step 4.
    # Since we are in a new context, we must use the values from step 4.
    # However, torch.autograd.grad returns tensors that require grad if retain_graph=True.
    # But we exited no_grad. We need to pass the gradients manually.
    
    # Re-calculate or store gradients from step 4? 
    # The source says: "Concatenate the cached gradients... Backpropagate the cached gradients"
    # In PyTorch, if we didn't save them, we can't. 
    # But in step 4, we computed them. We can store them.
    
    # Let's assume we stored grad_c_prime and grad_sigma_prime from step 4.
    # We need to gather them for the hard indices.
    grad_c_hard = grad_c_prime[hard_indices]
    grad_sigma_hard = grad_sigma_prime[hard_indices]
    
    # Backward pass with these gradients
    # We want to update the network parameters.
    # attributes' corresponds to (c_prime_hard, sigma_prime_hard)
    # We perform backward pass on these tensors with the specific gradients.
    
    # Note: The source algorithm implies a single backward call on the tuple.
    # torch.autograd.backward or .backward on a tuple requires a gradient argument.
    
    torch.autograd.backward(
        [c_prime_hard, sigma_prime_hard],
        grad_tensors=[grad_c_hard, grad_sigma_hard],
        retain_graph=False
    )
    
    return new_tau_hat
```

**Edge Cases and Pitfalls:**
*   **Framework Overhead:** Standard deep learning platforms (PyTorch, TensorFlow) do not benefit from simply masking out samples during backpropagation because they still backpropagate zero gradients for masked samples and retain compute/memory overhead. The two-forward-one-backward approach is required to avoid this [S1].
*   **Unknown Hardness:** It is impossible to know beforehand which samples are hard, so intermediate tensors cannot be selectively saved during the first forward pass. Inference mode is used to avoid saving them entirely for the initial $B$ samples [S1].
*   **GPU Utilization:** If $b$ shrinks excessively small, GPU utilization drops and no computational time benefit is gained over pruned samples. Additionally, if $B/b$ reaches a certain point, the first forward pass dictates max memory requirements, negating memory benefits [S1].

## Operating characteristics
*   **Time Complexity:** The backward pass has nearly twice the number of operations compared to the forward pass. By reducing the backward pass to only $b$ samples (where $b \ll B$), both compute time and memory usage are reduced [S1]. Average iteration time is reduced to ~53-54% of baseline when $B/b$ is ~10-15% [S1].
*   **Memory Complexity:** Memory usage is reduced by ~40% compared to the baseline (e.g., from ~1 GB to ~570 MB) because the computational graph is only built for the smaller batch $b$ [S1].
*   **Performance:** On Instant-NGP, the method provides a 1 dB improvement in PSNR per unit training time on average, or a 2x speedup to reach the same PSNR level [S1].
*   **Strengths:** Hyperparameter-free batch size adjustment; significant memory savings; faster convergence.
*   **Limitations:** Requires two forward passes per iteration; overhead from ray sampling and volume rendering remains unchanged; potential GPU underutilization if $b$ is too small [S1].

## Background
The method was introduced by J. Korhonen et al. in 2024 [S1]. It addresses the computational bottleneck of the backward pass in NeRF optimization, where resources are spent on processing already learned samples [S1]. It is implemented in a PyTorch reimplementation of Instant-NGP and evaluated on the Mip-NeRF-360 dataset and synthetic NeRF dataset [S1].

## Sources
[S1] Efficient NeRF Optimization -- Not All Samples Remain Equally Hard, https://arxiv.org/abs/2408.03193v1, fetched 2026-10-27
