# WildGaussians

Also called: Splatting in the Wild images with Appearance-conditioned Gaussians, SWAG

## Overview
WildGaussians (SWAG) is a 3D Gaussian Splatting method designed for "in-the-wild" 3D reconstruction from unconstrained photo collections characterized by transient objects, occlusions, and varying illumination [S1][S2, unofficial][S3]. It solves the problem of photometric inconsistency and transient artifacts that cause standard 3DGS to produce floaters or color-shifted renders in such settings [S3]. It sets itself apart from its siblings by explicitly modeling image-dependent appearance variations via a lightweight MLP and separating static scene geometry from transient occluders using a learnable opacity variation term, enabling the discarding of transient Gaussians to recover a clean static scene [S3]. Unlike [ClipGS](9546-clipgs.md) which uses CLIP features, SWAG relies on DINO features and direct color prediction [S1, S3].

## How it works
**Inputs and Outputs:**
*   **Inputs:** A set of uncalibrated or calibrated images $\{I_k\}$ of a scene.
*   **Outputs:** A set of 3D Gaussians representing the static scene, and optionally a set of transient Gaussians. The method also produces image-dependent color and opacity adjustments during rendering.

**Data Structures:**
1.  **3D Gaussians:** Standard 3DGS parameters: mean $\mu$, covariance $\Sigma$ (or scale $s$ and rotation $q$), base opacity $\alpha$, and base color coefficients (SH) $c$.
2.  **Per-Image Embedding:** A trainable vector $l_I \in \mathbb{R}^n$ for each training image $I$. Default size $n=24$ [S3].
3.  **Hash Grid Encoding:** A hash table encoding for Gaussian centers $x$.
    *   Hash table size $T = 2^{19}$ [S3].
    *   Finest resolution $N_{max} = 2048$ [S3].
    *   Coarsest resolution $N_{min} = 16$ [S3].
    *   Number of levels $L = 12$ [S3].
4.  **MLP $F_\theta$:** A 3-layer MLP with 64 hidden units.
    *   **Inputs:** Gaussian base color $c$, Hash Grid embedding of center $emb(x)$, Per-image embedding $l_I$.
    *   **Outputs:** Image-dependent color $c^I$ and opacity location parameter $\Delta \alpha^I$.

**Numbered Steps:**

1.  **Initialization:** Initialize 3D Gaussians from SfM points. Initialize per-image embeddings $l_I$ randomly. Initialize Hash Grid and MLP weights.
2.  **Embedding Computation:** For each Gaussian with center $x$, compute its hash grid embedding $emb(x)$ using the configured Hash Grid parameters [S3].
3.  **Appearance and Opacity Prediction:** For a given training image $I$ and a Gaussian $g$:
    *   Compute the image-dependent color $c^I$ and opacity location parameter $\Delta \alpha^I$ using the MLP:
        $$ (c^I, \Delta \alpha^I) = F_\theta(c, emb(x), l_I) $$
        where $c$ is the base color derived from SH coefficients [S3].
4.  **Opacity Variation Sampling:** Sample the opacity variation $\Delta \tilde{\alpha}^I$ from a Binary Concrete distribution to allow differentiable masking of transient objects:
    $$ \Delta \tilde{\alpha}^I = \text{sigmoid} \left[ \frac{1}{T_{emp}} \left( \log(|\Delta \alpha^I|) + \log(U) - \log(1-U) \right) \right] $$
    where $U \sim \text{Uniform}(0,1)$ and $T_{emp}$ is the temperature [S3].
5.  **Final Opacity Calculation:** Compute the final image-dependent opacity $\alpha^I$:
    $$ \alpha^I = \max(\alpha - \Delta \tilde{\alpha}^I, 0) $$
    where $\alpha$ is the base Gaussian opacity [S3].
6.  **Rendering:** Render the image using standard 3DGS splatting with the modified color $c^I$ and opacity $\alpha^I$.
7.  **Loss Computation:** Minimize the L1 loss combined with D-SSIM term between the rendered image and the ground truth image $I$ [S3].
8.  **Optimization:** Update 3D Gaussian parameters, MLP weights $\theta$, Hash Encoder parameters, and per-image embeddings $l_I$ via gradient descent [S3].
9.  **Transient Identification (Post-Training):** After training, identify transient Gaussians as those with non-zero variance of their opacity variation $\Delta \tilde{\alpha}^I$ across all training images:
    $$ \text{Var}(\Delta \tilde{\alpha}^I) \neq 0 $$
    Gaussians with $\text{Var}(\Delta \tilde{\alpha}^I) = 0$ are considered static [S3].
10. **Static Scene Extraction:** Discard transient Gaussians to keep only the static part of the scene [S3].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import torch
import torch.nn as nn
import torch.nn.functional as F

class WildGaussiansRenderer:
    def __init__(self, num_images, hash_table_size=2**19, n_levels=12, n_min=16, n_max=2048, embed_dim=24):
        """
        Initialize the SWAG model components.
        """
        self.num_images = num_images
        self.embed_dim = embed_dim
        
        # Per-image embeddings
        self.image_embeddings = nn.Embedding(num_images, embed_dim)
        
        # Hash Grid Encoding for Gaussian centers
        # Note: In practice, this uses a specialized CUDA kernel or library like tiny-cuda-nn.
        # Here we abstract the lookup logic.
        self.hash_grid = HashGrid(
            n_levels=n_levels,
            n_features_per_level=2,
            log2_hashmap_size=19, # 2^19
            resolution_min=n_min,
            resolution_max=n_max
        )
        
        # MLP for appearance and opacity variation
        # Input: c (3) + emb(x) (n_levels * 2) + l_I (24)
        # Output: c_I (3) + delta_alpha_loc (1)
        input_dim = 3 + (n_levels * 2) + embed_dim
        self.mlp = nn.Sequential(
            nn.Linear(input_dim, 64),
            nn.ReLU(),
            nn.Linear(64, 64),
            nn.ReLU(),
            nn.Linear(64, 4) # 3 for color, 1 for opacity location
        )
        
        # Temperature for Binary Concrete
        self.temperature = 0.1

    def forward(self, gaussians, image_idx, base_color_sh):
        """
        Forward pass for a batch of Gaussians for a specific image.
        
        Args:
            gaussians: Dict with 'means' (N, 3), 'opacities' (N, 1), 'scales' (N, 3), 'rotations' (N, 4)
            image_idx: Integer index of the current training image
            base_color_sh: SH coefficients for base color (N, 3)
            
        Returns:
            rendered_color, rendered_opacity
        """
        means = gaussians['means']
        opacities = gaussians['opacities']
        
        # 1. Get per-image embedding
        l_I = self.image_embeddings(torch.tensor(image_idx)) # (embed_dim,)
        
        # 2. Compute Hash Grid embedding for centers
        # emb(x) shape: (N, n_levels * 2)
        emb_x = self.hash_grid(means)
        
        # 3. Prepare MLP input
        # Concatenate base color, center embedding, and image embedding
        # Expand l_I to match batch size N
        l_I_expanded = l_I.unsqueeze(0).expand(means.shape[0], -1)
        
        mlp_input = torch.cat([base_color_sh, emb_x, l_I_expanded], dim=-1)
        
        # 4. MLP Output
        # (c_I, delta_alpha_loc)
        mlp_out = self.mlp(mlp_input)
        c_I = mlp_out[:, :3]
        delta_alpha_loc = mlp_out[:, 3:4]
        
        # 5. Sample opacity variation from Binary Concrete
        # U ~ Uniform(0, 1)
        U = torch.rand_like(delta_alpha_loc)
        
        # Concrete distribution sampling
        # log(|delta_alpha_loc| + eps) to avoid log(0)
        eps = 1e-6
        logit = (torch.log(torch.abs(delta_alpha_loc) + eps) + 
                 torch.log(U + eps) - 
                 torch.log(1 - U + eps))
        
        delta_alpha_tilde = torch.sigmoid(logit / self.temperature)
        
        # 6. Final Opacity
        # alpha_I = max(alpha - delta_alpha_tilde, 0)
        alpha_I = torch.clamp(opacities - delta_alpha_tilde, min=0.0)
        
        return c_I, alpha_I

    def identify_transient_gaussians(self, all_delta_alpha_tildes):
        """
        Identify transient Gaussians based on variance of opacity variation across images.
        
        Args:
            all_delta_alpha_tildes: Tensor of shape (num_images, num_gaussians)
            
        Returns:
            is_transient: Boolean tensor (num_gaussians,)
        """
        # Variance across images (dim=0)
        variance = torch.var(all_delta_alpha_tildes, dim=0)
        
        # Transient if variance is non-zero
        is_transient = variance > 1e-6
        
        return is_transient
```

**Parameters:**
*   **Temperature $T_{emp}$:** Controls the concentration of the Concrete distribution. Set to $0.1$ to force a binary distribution from the beginning of training. No annealing is used [S3].
*   **Evaluation $U$:** During evaluation/inference, $U$ is fixed at $0.5$ [S3].
*   **Latent embedding size $n$:** Size of per-image embedding vectors. Set to $24$ [S3].
*   **Hash Table Size:** $2^{19}$ [S3].
*   **Finest Resolution $N_{max}$:** $2048$ [S3].
*   **Number of Levels $L$:** $12$ [S3].
*   **Coarsest Resolution $N_{min}$:** $16$ [S3].

**Edge Cases and Pitfalls:**
*   **Over-construction:** The method may over-construct static parts of the scene, leading to artifacts such as black spots on roads (e.g., Brandenburg Gate scene) or incorrect rendering of fountain water (e.g., Trevi Fountain scene) [S3].
*   **Transient Detection:** In static scenes, a small percentage of Gaussians (<4%) may be incorrectly detected as transient. Removing them does not significantly affect quality, but care should be taken not to remove valid static geometry [S3].
*   **Affine Transformation Limitation:** Using affine color transformations instead of the MLP for appearance modeling fails to capture all appearance changes in in-the-wild scenarios [S3].

## Subtopics

### [Dynamic Appearance Features Modeling](3776-dynamic-appearance-features-modeling.md)
Addresses the limitation of global feature embeddings by extracting dynamic appearance features from a reference image and allowing each Gaussian to sample from multiple feature maps, capturing local details like highlights and shadows. Choose this when local appearance variations are more critical than global scene embeddings.

## Operating characteristics
*   **Time Complexity:** Training is one order of magnitude faster than previous implicit methods (NeRF-W, Ha-NeRF, CR-NeRF, RefinedFields) [S3]. Rendering speed matches real-time 3DGS [S1][S2, unofficial].
*   **Memory Complexity:** Uses more Gaussians than standard 3DGS due to less frequent densification steps but injection of image-dependent information [S3]. The number of transient Gaussians can be significant (e.g., 573,644 in Trevi Fountain vs 109,672 for 3DGS) [S3].
*   **Performance:** Improves 3DGS PSNR by an average margin of 5.01 dB on Phototourism scenes and 5 dB on NeRF-OSR benchmark [S3]. Outperforms baselines in SSIM and LPIPS [S3].
*   **Strengths:** Handles unconstrained photo collections, transient objects, and varying illumination. Enables controllable appearance via latent space interpolation [S3].
*   **Limitations:** Failure cases include artifacts in specific areas (black spots on roads, fountain water) [S3]. Requires careful handling of transient vs. static separation.

## Background
WildGaussians (SWAG) was introduced to address the limitations of 3DGS in "in-the-wild" settings, which often suffer from artifacts due to transient objects and photometric variations [S3]. It builds on the 3DGS framework [8] and incorporates ideas from previous works like Urban Radiance Fields [17] and NeRF-W [14]. The method was evaluated on the Phototourism dataset (Brandenburg Gate, Sacre Coeur, Trevi Fountain) and the NeRF-OSR benchmark [S3].

## Sources
[S1] From Volume Rendering to 3D Gaussian Splatting: Theory and Applications, https://doi.org/10.1109/sibgrapi67909.2025.11223276, fetched 2026-10-26
[S2] From Volume Rendering to 3D Gaussian Splatting, https://www.visgraf.impa.br/Data/RefBib/PS_PDF/tutorial-sib2025/tutorial-sib2025.pdf, fetched 2026-10-28 (unofficial)
[S3] SWAG: Splatting in the Wild images with Appearance-conditioned Gaussians, https://doi.org/10.48550/arXiv.2403.10427, fetched 2026-10-29
