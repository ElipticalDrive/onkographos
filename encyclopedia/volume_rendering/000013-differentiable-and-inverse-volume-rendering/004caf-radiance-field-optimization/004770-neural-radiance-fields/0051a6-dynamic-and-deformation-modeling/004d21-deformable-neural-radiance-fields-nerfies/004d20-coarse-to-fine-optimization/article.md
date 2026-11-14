# Coarse-to-Fine Optimization

Also called: Exponential Loss Weighting Strategy, Exponential Weight Decay for Coarse-to-Fine, Frequency Annealing for Deformation Fields

## Overview
Coarse-to-Fine Optimization is a training strategy for deformable neural radiance fields that stabilizes convergence by gradually increasing the model's capacity to represent high-frequency details. It addresses the difficulty of simultaneously optimizing complex geometry, texture, and non-rigid deformation from scratch by either decaying the weight of a coarse reconstruction loss relative to a fine-grained loss [S1] or by annealing the frequency components of the positional encoding used by the deformation network [S2].

## How it works
This method manifests in two distinct implementations found in the literature: one based on loss weighting (DP-NeRF) and one based on positional encoding modulation (Nerfies). Both aim to prevent the optimizer from getting stuck in local minima associated with high-frequency noise early in training.

### Inputs and Outputs
- **Inputs**: Training images, camera poses, latent codes (for deformation conditioning), and hyperparameters ($\lambda_s, \lambda_e, e_c, e_f$ for loss weighting; frequency cutoff schedule for encoding).
- **Outputs**: Updated network weights (deformation field, radiance field) and optimized scene representation.

### Data Structures
- **Positional Encoding Vector**: A vector $\phi(\mathbf{x})$ constructed by applying sinusoidal functions at multiple frequencies to input coordinates $\mathbf{x}$.
- **Loss Weights**: A scalar $\lambda$ that varies per training iteration.

### Method 1: Exponential Loss Weighting (DP-NeRF)
This approach balances two reconstruction losses: the Rigid Background Kernel (RBK) loss (coarse) and the Adaptive Weighting Prior (AWP) loss (fine).

1.  **Define Hyperparameters**:
    -   $\lambda_s$: Initial weight for the coarse loss (typically $> 1$).
    -   $\lambda_e$: Final weight for the coarse loss (typically $< 1$).
    -   $e_c$: Start epoch for decay.
    -   $e_f$: End epoch for decay.
    -   $e_i$: Current epoch.

2.  **Calculate Decay Rate**:
    Compute the exponential decay rate $\alpha$ using Equation 22 from [S1]:
    $$ \alpha = -\frac{\log(\lambda_s / \lambda_e)}{e_f - e_c} $$

3.  **Compute Current Weight**:
    Calculate the current weight $\lambda$ for the coarse loss at epoch $e_i$ using Equation 22 from [S1]:
    $$ \lambda = \lambda_s \exp(\alpha (e_c - e_i)) $$
    *Note: If $e_i < e_c$, $\lambda = \lambda_s$. If $e_i > e_f$, $\lambda = \lambda_e$.*

4.  **Compute Total Loss**:
    The total loss $\mathcal{L}_{total}$ is a weighted sum of the RBK loss ($\mathcal{L}_{RBK}$) and the AWP loss ($\mathcal{L}_{AWP}$):
    $$ \mathcal{L}_{total} = \lambda \mathcal{L}_{RBK} + (1 - \lambda) \mathcal{L}_{AWP} $$
    (Assuming normalized weights or specific scaling as per [S1]; the text specifies "exponentially decaying weight $\lambda$ ... between the two losses").

5.  **Backpropagate**:
    Update network weights using gradient descent on $\mathcal{L}_{total}$.

### Method 2: Frequency Annealing (Nerfies)
This approach modulates the input positional encoding of the deformation field network to restrict its capacity initially.

1.  **Initialize Positional Encoding**:
    Construct the standard positional encoding $\phi(\mathbf{x})$ for input coordinates $\mathbf{x}$ (typically 3D spatial coordinates + time/latent code).

2.  **Apply Frequency Mask**:
    At the start of optimization, zero out the components of $\phi(\mathbf{x})$ corresponding to high frequencies. This limits the deformation field to learning only smooth, low-frequency deformations.

3.  **Anneal Frequencies**:
    As optimization proceeds, gradually introduce higher frequency components into the positional encoding. This allows the network to refine the deformation field with finer details.

4.  **Backpropagate**:
    Update network weights. The effective capacity of the deformation field increases over time.

### Pseudocode (Loss Weighting Variant)
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def compute_coarse_to_fine_weight(current_epoch, start_epoch, end_epoch, lambda_start, lambda_end):
    """
    Computes the exponential decay weight lambda for coarse-to-fine optimization.
    Based on DP-NeRF [S1].
    """
    if current_epoch < start_epoch:
        return lambda_start
    if current_epoch > end_epoch:
        return lambda_end
    
    # Calculate decay rate alpha
    # alpha = -log(lambda_s / lambda_e) / (e_f - e_c)
    alpha = -math.log(lambda_start / lambda_end) / (end_epoch - start_epoch)
    
    # Calculate lambda = lambda_s * exp(alpha * (e_c - e_i))
    lambda_current = lambda_start * math.exp(alpha * (start_epoch - current_epoch))
    
    return lambda_current

def train_step(loss_rbk, loss_awp, current_epoch, start_epoch, end_epoch, lambda_start, lambda_end):
    """
    Computes total loss and returns gradients.
    """
    lambda_weight = compute_coarse_to_fine_weight(
        current_epoch, start_epoch, end_epoch, lambda_start, lambda_end
    )
    
    # Total loss is weighted sum of coarse (RBK) and fine (AWP) losses
    # Note: The exact normalization of the second term is not explicitly defined in [S1] beyond "balance"
    # Assuming standard weighted sum where lambda scales the coarse term
    total_loss = lambda_weight * loss_rbk + loss_awp
    
    return total_loss
```

### Pseudocode (Frequency Annealing Variant)
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def get_positional_encoding(x, max_freq_index, current_freq_index):
    """
    Returns positional encoding with only the first current_freq_index frequencies active.
    Based on Nerfies [S2].
    """
    # Standard positional encoding generates sin(2^i * pi * x) for i in range(max_freq_index)
    # Zero out frequencies above current_freq_index
    encoded = []
    for i in range(max_freq_index):
        if i < current_freq_index:
            # Include frequency component
            encoded.append(math.sin(2**i * math.pi * x))
        else:
            # Zero out high frequencies
            encoded.append(0.0)
    return encoded

def anneal_frequency_schedule(current_epoch, total_epochs, min_freq, max_freq):
    """
    Linearly increases the number of active frequencies from min_freq to max_freq.
    """
    progress = current_epoch / total_epochs
    current_freq = min_freq + progress * (max_freq - min_freq)
    return int(current_freq)
```

### Parameters and Edge Cases
-   **$\lambda_s, \lambda_e$**: Must be chosen such that $\lambda_s > \lambda_e$ for decay. If $\lambda_s = \lambda_e$, no annealing occurs.
-   **Frequency Cutoff**: Must be integer indices into the positional encoding vector.
-   **Precision**: Floating point precision in exponential calculations should be sufficient to avoid underflow for large epoch differences.
-   **Degenerate Input**: If $e_c = e_f$, the denominator in $\alpha$ is zero. This case is undefined; sources do not specify handling, implying $e_c < e_f$ is required [S1].

## Operating characteristics
-   **Time Complexity**: Adds negligible overhead to standard NeRF training. Loss weighting requires one extra multiplication and addition per step. Frequency annealing requires masking the positional encoding vector, which is $O(D)$ where $D$ is the encoding dimension.
-   **Memory Complexity**: No additional memory required beyond standard buffers.
-   **Strengths**: Prevents oscillation and local minima in high-frequency deformation fields. Stabilizes training for scenes with complex textures or non-rigid motion.
-   **Limitations**: Requires careful tuning of hyperparameters ($\lambda_s, \lambda_e, e_c, e_f$ or frequency schedule). May slow down convergence if the decay is too slow.
-   **When to Use**: When training deformable NeRFs (Nerfies) or deblurred NeRFs (DP-NeRF) from scratch. Not needed if using pre-trained weights or simpler rigid scenes.

## Background
Coarse-to-Fine Optimization was introduced in two key papers:
1.  **Nerfies** (Park et al., 2021) [S2]: Proposed frequency annealing of positional encodings to regularize the deformation field, preventing it from overfitting to high-frequency noise early in training.
2.  **DP-NeRF** (Zhang et al., 2022) [S1]: Proposed exponential decay of the weight between a coarse rigid background loss and a fine adaptive weighting prior loss to stabilize training for deblurred scene reconstruction.

Both methods address the instability of optimizing high-capacity neural networks for complex 3D representations from scratch.

## Sources
[S1] DP-NeRF: Deblurred Neural Radiance Field with Physical Scene Priors, https://arxiv.org/abs/2211.12046v4, fetched 2026-10-26
[S2] Nerfies: Deformable Neural Radiance Fields, https://doi.org/10.1109/iccv48922.2021.00581, fetched 2026-10-26
