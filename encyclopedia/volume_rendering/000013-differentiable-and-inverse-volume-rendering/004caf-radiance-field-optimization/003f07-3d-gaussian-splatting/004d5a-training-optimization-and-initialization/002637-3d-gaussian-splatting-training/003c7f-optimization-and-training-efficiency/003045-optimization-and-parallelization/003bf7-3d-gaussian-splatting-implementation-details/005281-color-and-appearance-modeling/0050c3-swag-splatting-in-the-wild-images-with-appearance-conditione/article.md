# SWAG (Splatting in the Wild images with Appearance-conditioned Gaussians)

## Overview
SWAG is an extension of 3D Gaussian Splatting (3DGS) designed to reconstruct 3D scenes from unstructured, in-the-wild image collections that exhibit significant photometric variations and contain transient occluders (e.g., tourists, clouds) [S1]. It differs from standard 3DGS by decoupling appearance modeling from static geometry: it uses per-image embeddings and a neural network to predict image-dependent color and opacity variations, while simultaneously identifying and isolating transient objects into separate "transient Gaussians" to prevent them from causing floaters or blurring the static scene reconstruction [S1]. Unlike [3iGS](13977-3igs-factorised-tensorial-illumination-for-3d-gaussian-splat.md), which focuses on factorized tensorial illumination, SWAG explicitly handles transient occlusion and appearance interpolation via latent embeddings [S1].

## How it works
SWAG retains the core 3DGS rasterization pipeline but modifies the color $\mathbf{c}_i$ and opacity $\alpha_i$ attributes of each Gaussian $i$ to be conditioned on the input image [S1]. The method optimizes a set of static Gaussian parameters alongside appearance-specific parameters to disentangle static scene structure from dynamic appearance and transient objects [S1].

### Inputs and Outputs
**Inputs:**
*   A set of unstructured images $\mathcal{I}$ with known camera poses.
*   Initial 3D point cloud (typically from SfM) to initialize Gaussian centroids.

**Outputs:**
*   Rendered images $\hat{\mathbf{C}}$ for novel views.
*   A disentangled scene representation separating static geometry, appearance variations, and transient objects.

### Data Structures
For each Gaussian $i$, the following parameters are optimized:
1.  **Static Attributes:**
    *   Centroid: $\mathbf{x}_i \in \mathbb{R}^3$
    *   Scale: $\mathbf{s}_i \in \mathbb{R}^3$
    *   Rotation: $\mathbf{q}_i \in \mathbb{R}^4$ (quaternion)
    *   Base Opacity: $\alpha_i \in \mathbb{R}$
    *   Base Color: $\mathbf{c}_i$ encoded as spherical harmonic (SH) coefficients [S1].
2.  **Appearance/Transient Attributes:**
    *   **Hash Grid Encoding:** A hash table $H$ maps 3D positions to feature vectors. The encoded center is $\text{emb}(\mathbf{x}_i) = H(\mathbf{x}_i)$ [S1].
    *   **Image Embedding:** A per-image latent vector $\mathbf{l}_I \in \mathbb{R}^{d_{emb}}$ for each training image $I$ [S1].
    *   **MLP Network ($F_\theta$):** A multi-layer perceptron with weights $\theta$ [S1].

**Hash Grid Configuration:**
*   Hash table size $T = 2^{19}$ [S1].
*   Coarsest resolution $N_{min} = 16$ [S1].
*   Finest resolution $N_{max} = 2048$ [S1].
*   Number of levels $L = 12$ [S1].

**MLP Configuration ($F_\theta$):**
*   Number of layers: 8 [S1].
*   Input dimension: 3 (concatenated base color, image embedding, and encoded center? Note: Source says input dim 3, but lists 3 inputs. See Pseudocode for clarification) [S1].
*   Hidden layer size: 51 [S1].
*   Output dimension: 64 (outputs $\mathbf{c}_I$ and $\Delta \alpha_I$) [S1].

### Numbered Steps

1.  **Initialization:** Initialize Gaussians from SfM points. Initialize Hash Grid $H$, per-image embeddings $\mathbf{l}_I$, and MLP weights $\theta$ [S1].
2.  **Feature Encoding:** For each Gaussian $i$ and training image $I$:
    *   Compute the spatial feature: $\mathbf{f}_{pos} = H(\mathbf{x}_i)$ [S1].
    *   Retrieve the image feature: $\mathbf{f}_{img} = \mathbf{l}_I$ [S1].
    *   Retrieve the base color: $\mathbf{f}_{color} = \mathbf{c}_i$ (decoded from SH) [S1].
3.  **Appearance and Opacity Prediction:**
    *   Concatenate inputs to form the MLP input vector. The source states the MLP takes "base color $\mathbf{c}$, image embedding $\mathbf{l}_I$, and encoded center $\text{emb}(\mathbf{x})$" [S1].
    *   Compute outputs:
        $$ [\mathbf{c}_I, \Delta \alpha_I] = F_\theta([\mathbf{c}_i, \mathbf{l}_I, \mathbf{f}_{pos}]) $$
        where $\mathbf{c}_I \in \mathbb{R}^3$ is the image-dependent color and $\Delta \alpha_I \in \mathbb{R}$ is the image-dependent opacity variation parameter [S1].
4.  **Opacity Modulation (Concrete Distribution):**
    *   To handle transient objects, the opacity variation $\Delta \alpha_I$ is treated as the location variable of a concrete distribution (Gumbel-Softmax style relaxation) to allow differentiable sampling [S1].
    *   Sample the opacity variation: $\Delta \tilde{\alpha}_I \sim \text{Concrete}(\Delta \alpha_I)$ [S1].
    *   Compute final opacity: $\alpha_{final} = \alpha_i \cdot \Delta \tilde{\alpha}_I$ (or similar modulation, source implies $\Delta \tilde{\alpha}_I$ modulates opacity) [S1].
    *   *Note:* The source states $\Delta \alpha_I$ is the location variable of a concrete distribution which is sampled to get $\Delta \tilde{\alpha}_I$ [S1].
5.  **Rendering:**
    *   Rasterize Gaussians using the standard 3DGS pipeline with the modified color $\mathbf{c}_I$ and opacity $\alpha_{final}$ [S1].
    *   The final pixel color is computed as:
        $$ \hat{\mathbf{C}} = \sum_{i \in N} T_i g_i(\mathbf{u} | \boldsymbol{\mu}', \boldsymbol{\Sigma}') \alpha_{final, i} \mathbf{c}_I $$
        where $T_i$ is transmittance and $g_i$ is the 2D Gaussian density [S1].
6.  **Transient Detection and Disentanglement:**
    *   After training, Gaussians are classified as static or transient based on the variance of the sampled opacity variation $\Delta \tilde{\alpha}_I$ across training images [S1].
    *   A Gaussian is considered transient if its opacity varies significantly across different images [S1].
    *   Static Gaussians are used for the final static scene reconstruction. Transient Gaussians can be removed or rendered separately [S1].

### Pseudocode

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import torch
import torch.nn as nn
import torch.nn.functional as F

class SWAG:
    def __init__(self, num_images, hash_table_size=2**19, n_min=16, n_max=2048, num_levels=12, 
                 emb_dim=24, mlp_hidden=51, mlp_layers=8, mlp_input_dim=3):
        """
        Initialize SWAG model components.
        """
        # 1. Hash Grid Encoding for Gaussian centers
        # Simplified HashGrid implementation details not provided in source.
        # Assumes a standard 3D HashGrid encoder mapping 3D coords to feature vectors.
        self.hash_grid = HashGrid(
            n_levels=num_levels,
            n_features_per_level=4, # Standard for such grids, not explicitly given but implied by output dim 64 and layers
            log2_hashmap_size=19, # 2^19
            resolution_min=n_min,
            resolution_max=n_max
        )
        
        # 2. Per-image embeddings
        # Shape: (num_images, emb_dim)
        self.image_embeddings = nn.Embedding(num_images, emb_dim)
        
        # 3. MLP for appearance and opacity prediction
        # Input: [base_color (3), image_emb (24), hash_feat (3?)] -> Source says input dim 3.
        # This is ambiguous. Source says "MLP takes three inputs... outputs two values".
        # And "Input dimension: 3". This likely refers to the number of *types* of inputs, 
        # or the source text has a typo regarding the concatenated dimension.
        # However, standard practice concatenates features. 
        # Let's assume the MLP input is the concatenation of the relevant features.
        # If input_dim=3 is literal, it might mean the MLP processes 3 scalar values? 
        # Unlikely given hidden size 51. 
        # Re-reading: "MLP Fθ Input dimension: 3". 
        # It is highly probable this refers to the number of input *vectors* concatenated, 
        # or it's a specific architectural choice. 
        # Given "outputs... cI and dalphaI", and cI is color (3D), dalpha is scalar.
        # Let's assume the input to the MLP is the concatenation of:
        # 1. Base color (3)
        # 2. Image embedding (24)
        # 3. Hash grid feature (variable, but let's assume it's projected or the "3" refers to something else).
        # Actually, looking at "Input dimension: 3" and "Hidden layers size: 51", 
        # and "Number of layer: 8".
        # If input is truly 3, it's very small. 
        # Let's look at the text: "MLP takes as inputs the Gaussians’ colors c, the associated image embedding lI, and their encoded centers emb(x)".
        # These are 3 inputs. Their dimensions are 3, 24, and HashFeat.
        # If HashFeat is not 3, the total input dim > 3.
        # Hypothesis: The "Input dimension: 3" in the table might refer to the number of *layers* before hidden? No, "Number of layer: 8".
        # Or maybe the hash grid output is projected to 3?
        # Or maybe the "3" is a typo for the number of *types* of inputs.
        # Given the ambiguity, I will implement the concatenation of the three specified inputs.
        # I will assume the HashGrid output is flattened or projected to match the expected input size if needed, 
        # but since the source doesn't specify the hash feature dimension, I will leave it as a variable `hash_feat_dim`.
        # However, to make it "valid Python" and self-contained, I must define it.
        # I will assume a standard hash grid feature dimension of 4 per level * levels? Or just 4?
        # Let's assume the input to the MLP is the concatenation of [c, lI, emb(x)].
        # I will define the MLP input size dynamically based on the components.
        
        self.mlp = nn.Sequential(
            nn.Linear(3 + emb_dim + 4, 51), # Assuming hash feat dim 4 for simplicity, or generic
            *[nn.Sequential(nn.ReLU(), nn.Linear(51, 51)) for _ in range(6)], # 8 layers total: 1 input + 6 hidden + 1 output? 
            # Source says "Number of layer: 8". Usually includes output.
            # Let's do 7 hidden + 1 output.
            nn.Linear(51, 64) # Output dim 64
        )
        
        # Note: The source specifies "Input dimension: 3". This is extremely specific.
        # If the input is strictly 3, then c, lI, and emb(x) must be reduced to 3 scalars?
        # Or maybe c is 3, and lI and emb(x) are not inputs? No, text says they are.
        # I will follow the text description of inputs over the table's "3" if it conflicts, 
        # but mark the discrepancy. 
        # Actually, looking at similar papers, "Input dimension" often refers to the SH degree or similar.
        # Here, it might be a typo for the number of *vectors* concatenated.
        # I will implement the concatenation of the three described inputs.

    def forward(self, gaussians, image_idx):
        """
        gaussians: dict with keys 'x', 'c' (SH coeffs), 'alpha', etc.
        image_idx: tensor of image indices for each Gaussian (broadcasted or per-pixel?)
        """
        # 1. Get Base Color from SH
        # Simplified: assume c is already the base color vector (3,)
        c_base = gaussians['c'] # Shape (N, 3)
        
        # 2. Get Image Embedding
        # image_idx shape (N,)
        l_I = self.image_embeddings(image_idx) # Shape (N, 24)
        
        # 3. Get Hash Grid Encoding
        x = gaussians['x'] # Shape (N, 3)
        emb_x = self.hash_grid(x) # Shape (N, D_hash)
        
        # 4. Concatenate Inputs
        # Assuming D_hash is known or projected. 
        # To satisfy "Input dimension: 3" literally is impossible with 3 vectors of dim >1.
        # I will assume the table's "3" is an error or refers to something else, 
        # and follow the textual description of inputs.
        mlp_input = torch.cat([c_base, l_I, emb_x], dim=-1)
        
        # 5. MLP Prediction
        # Output dim 64. Split into c_I (3) and d_alpha (1)? 
        # Source says outputs "cI and dalphaI". 
        # 64 is likely a bottleneck or intermediate. 
        # Let's assume the last layer outputs 4 (3 for color, 1 for opacity).
        # But table says Output dimension 64.
        # Maybe 64 is the hidden size? No, "Hidden layers size: 51".
        # Maybe the output is 64, and we extract parts?
        # I will assume the MLP outputs 4 values: 3 for color, 1 for opacity.
        # If the source insists on 64, it might be a latent code.
        # However, "outputs two values: an image-dependent color cI and ... dalphaI".
        # I will map the first 3 to cI and the next 1 to dalphaI.
        
        mlp_out = self.mlp(mlp_input) # Shape (N, 64)
        
        # Extract c_I and d_alpha_I
        # Assuming the first 3 channels are color and the 4th is opacity variation
        c_I = mlp_out[:, :3]
        d_alpha_I = mlp_out[:, 3:4]
        
        # 6. Concrete Distribution Sampling for Opacity
        # Sample from Concrete distribution with location d_alpha_I
        # Using Gumbel-Softmax approximation
        temperature = 1.0 # Not specified, typical value
        # Sample noise
        u1 = torch.rand_like(d_alpha_I)
        u2 = torch.rand_like(d_alpha_I)
        # Gumbel noise
        g1 = -torch.log(-torch.log(u1 + 1e-10) + 1e-10)
        g2 = -torch.log(-torch.log(u2 + 1e-10) + 1e-10)
        # Concrete sample
        # Standard concrete: sigmoid((log(u1) - log(1-u1) - log(u2) + log(1-u2) + loc) / temp)
        # Simplified:
        d_alpha_tilde = torch.sigmoid((d_alpha_I + g1 - g2) / temperature)
        
        # 7. Final Opacity
        alpha_base = gaussians['alpha']
        alpha_final = alpha_base * d_alpha_tilde
        
        return c_I, alpha_final

    def get_transient_mask(self, gaussians, image_indices, num_images):
        """
        Identify transient Gaussians based on variance of d_alpha_tilde across images.
        """
        # This requires forward pass over all training images for each Gaussian
        # to compute variance.
        # Simplified logic:
        # 1. Render/Compute d_alpha_tilde for all training images.
        # 2. Compute variance across images for each Gaussian.
        # 3. Threshold.
        pass
```

### Parameters
*   **Iterations:** 60,000 [S1].
*   **Densification Interval:** 2,000 [S1].
*   **Densification End:** 30,000 [S1].
*   **Image Embedding Dimension:** 24 [S1].
*   **Learning Rate (Image Embeddings):** 0.0025 [S1].
*   **Hash Grid Size:** $2^{19}$ [S1].
*   **Hash Grid Resolutions:** $N_{min}=16, N_{max}=2048$ [S1].
*   **Hash Grid Levels:** 12 [S1].
*   **Learning Rate (Hash Grid):** 0.0025 [S1].
*   **MLP Layers:** 8 [S1].
*   **MLP Hidden Size:** 51 [S1].
*   **MLP Learning Rate:** 0.0025 [S1].
*   **MLP Output Dimension:** 64 [S1].

### Edge Cases and Pitfalls
*   **Transient Detection Threshold:** Removing all Gaussians with zero variance in opacity variation ($\text{Var}(\Delta \tilde{\alpha}_I) < \lambda$) can leave holes in areas with frequent transients (e.g., clouds, tourists) [S1].
*   **Hole Filling:** To mitigate holes, an adaptive threshold $\lambda$ can be used instead of strict zero variance, but this requires manual tuning [S1].
*   **Performance:** Training time is almost double that of 3DGS; inference time per frame is 10 times longer than 3DGS [S1].

## Subtopics

### SWAG-A (Appearance-only)
Variant without transient detection. Enables appearance modeling and better colors than 3DGS, but transient objects still cause floaters [S1].

### SWAG-T (Transient-only)
Variant without appearance color dependency. Eliminates floaters and produces occluder-free scenes, but suffers from color alterations (average color convergence) like 3DGS [S1].

## Operating characteristics
*   **Time Complexity:** Training time is approximately 2x that of 3DGS [S1].
*   **Memory Complexity:** Increased due to Hash Grid, Image Embeddings, and MLP parameters [S1].
*   **Inference Speed:** 10x slower than 3DGS per frame, but still real-time [S1].
*   **Strengths:** Handles unstructured in-the-wild data; disentangles static and transient objects; enables appearance interpolation [S1].
*   **Limitations:** Slower training and inference than 3DGS; requires manual tuning for transient removal threshold to avoid holes [S1].
*   **When to use:** When reconstructing scenes from unstructured photo collections with significant lighting changes and transient occluders [S1].
*   **When not to use:** When real-time performance is critical and the scene is static and well-lit (use standard 3DGS) [S1].

## Background
SWAG was introduced by H. Dahmani et al. in 2024 [S1]. It addresses the limitations of 3DGS on unstructured in-the-wild data, such as the Phototourism and NeRF-OSR datasets, which contain illumination variations and transient objects [S1]. The method builds on the 3DGS framework but adds appearance conditioning and transient handling [S1].

## Sources
[S1] SWAG: Splatting in the Wild images with Appearance-conditioned Gaussians, https://doi.org/10.48550/arXiv.2403.10427, fetched 2026-10-29
