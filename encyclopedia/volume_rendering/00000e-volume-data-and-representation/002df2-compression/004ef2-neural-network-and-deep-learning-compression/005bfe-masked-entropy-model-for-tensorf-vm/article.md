# Masked Entropy Model for TensoRF-VM

Also called: ECTensoRF, Entropy Coded TensoRF, TensoRF-VM compression

## Overview
Masked Entropy Model for TensoRF-VM (ECTensoRF) is a compression method for TensoRF-VM feature planes that uses learnable binary masks to selectively compress only high-importance grid locations, addressing the inefficiency of standard entropy models on sparse, zero-heavy latent codes [S1, unofficial]. It sparsifies a pretrained TensoRF-VM model by optimizing binary masks over its latent tensors to minimize a rate-distortion loss [S1, unofficial]. Unlike standard entropy models that treat all latent values equally, ECTensoRF explicitly models the spatial sparsity of the feature grid, allowing it to achieve smaller file sizes with comparable or superior reconstruction quality compared to vector quantization baselines like VQ-TensoRF [S1, unofficial].

## How it works
**Inputs and Outputs:**
*   **Input:** A pretrained TensoRF-VM model consisting of three feature planes $P_i \in \mathbb{R}^{C_i \times W_i \times H_i}$ for $i \in \{1, 2, 3\}$, and a rendering function that uses these planes to generate images [S1, unofficial].
*   **Output:** A compressed model consisting of latent codes $Z_i$, binary masks $M_i$, and the parameters of a lightweight decoder $D$ and entropy model $P$ [S1, unofficial].

**Data Structures:**
1.  **Feature Plane ($P_i$):** The original high-dimensional grid from the pretrained TensoRF-VM model. Size: $C_i \times W_i \times H_i$ [S1, unofficial].
2.  **Latent Code ($Z_i$):** The compressed representation of $P_i$. Size: $C_{Z_i} \times (W_i/4) \times (H_i/4)$ [S1, unofficial].
3.  **Binary Mask ($M_i$):** A binary tensor of the same spatial dimensions as the latent code $Z_i$ (i.e., $C_{Z_i} \times (W_i/4) \times (H_i/4)$), where $M_{ji} \in \{0, 1\}$ [S1, unofficial].
4.  **Weight Map ($W_i$):** A spatial map of size $W_i \times H_i$ shared across channels, derived from importance scores [S1, unofficial].
5.  **Decoder ($D$):** A two-layer transposed convolutional neural network with SELU activation, kernel size 3, stride 2, and padding 1, providing an upsampling factor of 2 per layer [S1, unofficial].
6.  **Entropy Model ($P$):** A fully factorized probability distribution (spike-and-slab prior) parameterized by the mask probabilities $\pi^M$ [S1, unofficial].

**Algorithm Steps:**

1.  **Calculate Importance and Weight Maps:**
    Compute the importance scores $I_i$ for each feature plane (using the method from Li et al. [S1, unofficial]). Then, compute the weight maps $W_i$ using the log-transform and normalization:
    $$ W_i = \text{normalize}(\log(I_i + \epsilon)) $$
    where $\epsilon = 0.01$ [S1, unofficial]. The `normalize` function scales the values to the range $[0, 1]$ [S1, unofficial].

2.  **Initialize Parameters:**
    *   Initialize latent codes $Z_i$ as zero-tensors (or Gaussian random tensors; zero and Gaussian initialization produce nearly identical PSNR and file size results across all $\lambda$ values [S1, unofficial]).
    *   Initialize the decoder $D$ and entropy model $P$ with mask parameters $\{\pi^{M_i}\}_{i=1}^3$ [S1, unofficial].

3.  **Training Loop:**
    While not converged:
    a.  **Sample Masks:** Sample binary masks $M_i$ using the straight-through Gumbel-Softmax estimator. For each element $j$ in mask $M_i$:
        $$ M_{ji} = \arg\max_{k \in \{0,1\}} (g_{ji} + \log \pi^{M_i}_k) $$
        where $g_{ji} \sim \text{Gumbel}(0, 1)$ are i.i.d. samples [S1, unofficial].
    b.  **Apply Masks:** Compute the masked latent code $\hat{Z}_i$:
        $$ \hat{Z}_i = Z_i \odot M_i $$
        (Element-wise multiplication. If $M_{ji}=0$, then $\hat{Z}_{ji}=0$) [S1, unofficial].
    c.  **Reconstruct Feature Planes:** Pass the masked latent code through the decoder:
        $$ \hat{P}_i = D(\hat{Z}_i) $$
        [S1, unofficial].
    d.  **Render:** Render the scene using the reconstructed feature planes $\{\hat{P}_i\}_{i=1}^3$ [S1, unofficial].
    e.  **Calculate Loss:** Compute the rate-distortion loss $L$:
        $$ L = \mathcal{L}_{\text{render}}(\{\hat{P}_i\}_{i=1}^3) + \sum_{i=1}^3 \|(P_i - \hat{P}_i) \odot W_i\|_2^2 - \lambda \log_2 P_{M_i}(\hat{Z}_i) $$
        where $\mathcal{L}_{\text{render}}$ is the NeRF rendering loss, the second term is the importance-weighted feature plane reconstruction loss, and the third term is the rate loss defined as the log-likelihood of the entropy model $P$ [S1, unofficial].
    f.  **Update:** Update model parameters (latent codes $Z_i$, decoder $D$, entropy model $P$, and mask probabilities $\pi^M$) via backpropagation [S1, unofficial].

4.  **Annealing:**
    Use an annealing softmax temperature $\tau$ for the Gumbel-Softmax sampling, decaying from 10 to 0.1 during training [S1, unofficial].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import torch
import torch.nn.functional as F

def gumbel_softmax_straight_through(logits, temperature):
    """
    Samples binary masks using straight-through Gumbel-Softmax.
    logits: tensor of shape (..., 2) representing log(pi_0, pi_1)
    temperature: scalar tau
    Returns: hard binary samples (0 or 1) with straight-through gradients
    """
    gumbels = -torch.log(-torch.log(torch.rand_like(logits) + 1e-9)) + logits
    y_soft = F.softmax(gumbels / temperature, dim=-1)
    # Straight-through: use argmax for forward, soft for backward
    index = y_soft.max(dim=-1, keepdim=True)[1]
    y_hard = torch.zeros_like(y_soft).scatter_(-1, index, 1.0)
    y = y_hard - y_soft.detach() + y_soft
    return y

def normalize_log_importance(I, epsilon=0.01):
    """
    Computes weight map W from importance map I.
    I: tensor of shape (H, W)
    """
    log_I = torch.log(I + epsilon)
    min_val = log_I.min()
    max_val = log_I.max()
    if max_val - min_val < 1e-9:
        return torch.zeros_like(log_I)
    return (log_I - min_val) / (max_val - min_val)

class TransposedConvDecoder(torch.nn.Module):
    def __init__(self, in_channels, hidden_channels, out_channels):
        super().__init__()
        self.conv1 = torch.nn.ConvTranspose2d(in_channels, hidden_channels, kernel_size=3, stride=2, padding=1)
        self.conv2 = torch.nn.ConvTranspose2d(hidden_channels, out_channels, kernel_size=3, stride=2, padding=1)
        self.activation = torch.nn.SELU()

    def forward(self, x):
        x = self.activation(self.conv1(x))
        x = self.activation(self.conv2(x))
        return x

class MaskedEntropyModel:
    def __init__(self, latent_channels, feature_channels, width, height, lambda_rate=1e-4):
        self.lambda_rate = lambda_rate
        self.epsilon = 0.01
        self.tau_start = 10.0
        self.tau_end = 0.1
        
        # Decoder: 2 transposed conv layers, kernel 3, stride 2, padding 1, SELU
        self.decoder = TransposedConvDecoder(latent_channels, 192, feature_channels)
        
        # Entropy model parameters (log-probabilities for mask being 0 or 1)
        # Shape: (C_z, W/4, H/4, 2)
        self.mask_logits = torch.nn.Parameter(torch.zeros(latent_channels, width // 4, height // 4, 2))
        
        # Latent codes Z: 3 tensors, one per feature plane
        self.latents = [torch.nn.Parameter(torch.zeros(latent_channels, width // 4, height // 4)) for _ in range(3)]
        
        # Precomputed importance maps and weights (from pretrained TensoRF-VM)
        self.weight_maps = []

    def compute_weight_maps(self, importance_maps):
        """
        importance_maps: list of 3 tensors, each (H, W)
        """
        self.weight_maps = [normalize_log_importance(I, self.epsilon) for I in importance_maps]

    def sample_masks(self, temperature):
        """
        Samples binary masks using straight-through Gumbel-Softmax.
        Returns: list of 3 binary masks, each (C_z, W/4, H/4)
        """
        masks = []
        for _ in range(3):
            flat_logits = self.mask_logits.view(-1, 2)
            samples = gumbel_softmax_straight_through(flat_logits, temperature)
            indices = samples.argmax(dim=-1)
            indices = indices.view(self.mask_logits.shape[0], self.mask_logits.shape[1], self.mask_logits.shape[2])
            masks.append(indices.float())
        return masks

    def reconstruct(self, masks):
        """
        Reconstructs feature planes from latents and masks.
        masks: list of 3 tensors, each (C_z, W/4, H/4)
        Returns: list of 3 reconstructed feature planes, each (C_f, W, H)
        """
        recon_planes = []
        for i in range(3):
            masked_latent = self.latents[i] * masks[i]
            masked_latent = masked_latent.unsqueeze(0)  # Add batch dim for ConvTranspose2d
            recon = self.decoder(masked_latent)
            recon_planes.append(recon.squeeze(0))
        return recon_planes

    def calculate_loss(self, recon_planes, original_planes, masks, render_loss_fn):
        """
        Calculates the rate-distortion loss.
        """
        # 1. Rendering Loss
        loss_render = render_loss_fn(recon_planes)
        
        # 2. Weighted Reconstruction Loss
        loss_recon = 0.0
        for i in range(3):
            W_i = self.weight_maps[i].unsqueeze(0).expand_as(original_planes[i])
            diff = original_planes[i] - recon_planes[i]
            loss_recon += torch.sum((diff * W_i) ** 2)
            
        # 3. Rate Loss: -lambda * log2 P_Mi(Zi)
        # Compute log-probability of the sampled mask configuration
        log_probs = F.log_softmax(self.mask_logits, dim=-1)
        rate_loss = 0.0
        for i in range(3):
            mask_indices = masks[i].long().unsqueeze(-1)
            log_p_mask = log_probs.gather(-1, mask_indices).squeeze(-1)
            rate_loss += torch.sum(log_p_mask)
            
        total_loss = loss_render + loss_recon - self.lambda_rate * rate_loss
        return total_loss

    def train_step(self, original_planes, render_loss_fn, iteration, total_iterations):
        # Anneal temperature
        tau = self.tau_start - (self.tau_start - self.tau_end) * (iteration / total_iterations)
        
        # Sample masks
        masks = self.sample_masks(tau)
        
        # Reconstruct
        recon_planes = self.reconstruct(masks)
        
        # Calculate loss
        loss = self.calculate_loss(recon_planes, original_planes, masks, render_loss_fn)
        
        # Backprop
        loss.backward()
        # Optimizer.step() would go here
        return loss
```

**Parameters:**
*   $\lambda$: Rate-distortion trade-off parameter. Values experimented: $\{0.02, 0.01, 0.005, 0.001, 0.0005, 0.0002, 0.0001\}$. Higher $\lambda$ signifies a more compact model [S1, unofficial].
*   $\epsilon$: Epsilon added to importance maps for log stability. Value: $0.01$ [S1, unofficial].
*   $\tau$: Softmax temperature for Gumbel-Softmax. Anneals from $10$ to $0.1$ [S1, unofficial].
*   Latent Initialization: Zero-tensors or Gaussian random tensors. Zero and Gaussian initialization produce nearly identical PSNR and file size results across all $\lambda$ values [S1, unofficial].

**Edge Cases and Pitfalls:**
*   **Sparsity Inefficiency:** Standard entropy models fail to capture the sparsity in latent codes (predominantly zero, especially in background), leading to inefficiencies [S1, unofficial].
*   **Non-differentiable Masks:** Binary masks cannot be learned directly. Solved by using the Gumbel-Softmax estimator with straight-through gradients [S1, unofficial].
*   **Hyperprior Overhead:** Using a hyperprior model introduces a fixed overhead cost that degrades compression efficiency at very low bit rates, as the hyper decoder and entropy model parameters must be transmitted regardless of scene complexity [S1, unofficial].

## Subtopics

### Hyperprior Variant
An optional entropy modeling variant that applies masking on both hyper-latents and latents, optimized directly without amortized inference. The hyper decoder consists of two transposed convolutional layers with SELU activation. It outperforms the baseline at high bit rates but performs worse at low bit rates due to irreducible overhead [S1, unofficial].

### Two-Stage vs. End-to-End Training
Two-stage training (pretrain TensoRF, then compress) consistently outperforms end-to-end training (compress from scratch) across all $\lambda$ values, yielding higher PSNR for the same file size [S1, unofficial].

## Operating characteristics
*   **Time and Memory Complexity:** The method requires training time proportional to the number of iterations (30,000 in experiments) and the complexity of the decoder and entropy model. Memory usage is dominated by the latent codes and the pretrained TensoRF-VM model during training [S1, unofficial].
*   **Measured Performance:**
    *   On Synthetic-NeRF, ECTensoRF-H achieves 32.31 PSNR and 0.956 SSIM at 1.6 MB, compared to VQ-TensoRF's 32.86 PSNR/0.960 SSIM at 3.6 MB [S1, unofficial].
    *   At $\lambda=1e-4$, two-stage training achieves 33.00 dB PSNR at 4.24 MB, while end-to-end training achieves 31.80 dB at 2.95 MB [S1, unofficial].
    *   ECTensoRF-L shows superior reconstruction performance (PSNR/SSIM) with reduced file size on Synthetic-NeRF, Synthetic-NSVF, and Tanks & Temples. On LLFF, SSIM surpasses baselines, and file size is half of VQ-TensoRF [S1, unofficial].
*   **Scaling:** The method scales with the resolution of the feature planes. The decoder upsamples by a factor of 4 [S1, unofficial].
*   **Strengths:** Significant compression ratios (often <2MB for high quality), superior rate-distortion performance compared to VQ-TensoRF and Re:TensoRF, ability to selectively compress high-importance regions [S1, unofficial].
*   **Limitations:** Requires a pretrained TensoRF-VM model (two-stage training is recommended), hyperprior variant has fixed overhead, performance on LLFF dataset is slightly lower in PSNR compared to baselines (though SSIM is better) [S1, unofficial].

## Background
The method was introduced in the paper "Neural NeRF Compression" by Yang & Mandt (2023b) [S1, unofficial]. It builds on the TensoRF-VM representation [S1, unofficial] and addresses the limitations of standard neural image compression techniques when applied to the sparse latent codes of NeRFs. The approach leverages the spatial sparsity of feature grids and uses importance-weighted training to guide the compression process [S1, unofficial].

## Sources
[S1] Neural NeRF Compression, https://doi.org/10.48550/arxiv.2406.08943, fetched 2026-10-26 (unofficial)
