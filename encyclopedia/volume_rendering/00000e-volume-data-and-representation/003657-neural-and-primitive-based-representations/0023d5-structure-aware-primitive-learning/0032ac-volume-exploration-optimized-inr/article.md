# Volume Exploration-Optimized INR

Also called: Dual-pathway INR, FiLM-modulated INR

## Overview
Volume Exploration-Optimized INR is a dual-pathway Implicit Neural Representation architecture designed to learn voxel-level feature representations for volume exploration tasks, specifically image-based transfer function design [S1, unofficial]. It solves the problem of capturing local structural context and spatial coherence in scientific simulation data, which standard INRs often miss by relying solely on global positional encoding [S1, unofficial]. It sets itself apart from standard INRs by combining a multi-resolution hash grid for positional encoding with a local structural encoder, fused via Feature-wise Linear Modulation (FiLM), to jointly estimate intensity, gradient, and local statistics [S1, unofficial].

## How it works
**Inputs and Outputs:**
*   **Input:** A 3D coordinate $v = (x, y, z)$ and a raw volume dataset $V$ [S1, unofficial].
*   **Output:** A voxel-level feature vector (activations from the final hidden layer of the main MLP) and, during training, estimates for intensity, gradient, local mean, and local standard deviation [S1, unofficial].

**Data Structures:**
1.  **Multi-resolution Hash Grid:** Maps 3D coordinates to trainable features at multiple spatial resolutions [S1, unofficial].
2.  **Structural Encoder:** A Multi-Layer Perceptron (MLP) with two hidden layers, each containing 32 channels [S1, unofficial].
3.  **Main MLP:** A Multi-Layer Perceptron (MLP) with four hidden layers, each containing 64 channels [S1, unofficial].
4.  **Local Patch:** A cubic voxel patch of size $n^3$ centered at the query coordinate, where $n=5$ [S1, unofficial].

**Numbered Steps:**

1.  **Positional Encoding (Pathway 1):**
    Feed the 3D coordinate $v$ into the multi-resolution hash grid encoding to produce the positional representation $F_{pos}$ [S1, unofficial].

2.  **Structural Encoding (Pathway 2):**
    a. Sample a local patch of $n^3$ voxels from the raw volume $V$, centered at $v$, where $n=5$ [S1, unofficial].
    b. Process this patch through the structural encoder (two hidden layers, 32 channels each) to generate a structural representation $F_{struct}$ [S1, unofficial].

3.  **Feature Modulation (FiLM):**
    Pass $F_{struct}$ through a linear layer to generate channelwise scale $\gamma$ and shift $\beta$ parameters [S1, unofficial].
    Compute the modulated feature $F_{mod}$ using the formula:
    $$ F_{mod} = F_{pos} \odot (\gamma + 1) + \beta $$
    where $\odot$ denotes element-wise multiplication [S1, unofficial]. An offset of 1 is applied to $\gamma$ to maintain an identity mapping at initialization and improve numerical stability [S1, unofficial].

4.  **Main MLP Forward Pass:**
    Feed $F_{mod}$ into the main MLP (four hidden layers, 64 channels each) [S1, unofficial]. Utilize a residual connection between the modulated input $F_{mod}$ and the deeper layers of the MLP to facilitate gradient flow [S1, unofficial].

5.  **Multi-task Learning (Training):**
    The main MLP jointly estimates four targets: intensity, gradient, local mean, and local standard deviation [S1, unofficial].
    The total loss $L_{total}$ is defined as:
    $$ L_{total} = L_{inten} + \lambda_{grad} L_{grad} + \lambda_{stat} L_{\mu} + L_{\sigma} $$
    where $L_{inten}$, $L_{grad}$, $L_{\mu}$, and $L_{\sigma}$ are mean squared error losses for intensity, gradient, local mean, and local standard deviation, respectively, and $\lambda_{grad}$, $\lambda_{stat}$ are weighting hyperparameters [S1, unofficial].

6.  **Feature Extraction (Post-training):**
    Use the activations from the final hidden layer of the main MLP at each coordinate as the voxel-level feature representation for downstream volume exploration tasks (e.g., transfer function design) [S1, unofficial].

**Parameters:**
*   Patch size $n$: 5 [S1, unofficial].
*   Structural encoder layers: 2 hidden layers, 32 channels each [S1, unofficial].
*   Main MLP layers: 4 hidden layers, 64 channels each [S1, unofficial].
*   Optimizer: Adam [S1, unofficial].
*   Learning rate: $1e-4$ [S1, unofficial].
*   Training epochs: 100 [S1, unofficial].

**Edge Cases and Pitfalls:**
*   **Initialization:** The offset of 1 in the FiLM modulation ($\gamma + 1$) is critical to maintain identity mapping at initialization [S1, unofficial].
*   **Memory:** High-resolution volumes may exceed memory limits for implicit convolutional feature-based methods, requiring high-RAM servers (e.g., 512 GB RAM) [S1, unofficial].
*   **Thresholding:** In downstream classification tasks, voxels with probability values less than 0.5 are considered background and excluded [S1, unofficial].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import torch
import torch.nn as nn
import torch.nn.functional as F

class VolumeExplorationOptimizedINR(nn.Module):
    def __init__(self, hash_grid, raw_volume, n=5):
        """
        Args:
            hash_grid: Multi-resolution hash grid encoder instance.
            raw_volume: Tensor of shape (D, H, W) containing the raw volume data.
            n: Size of the local patch dimension (default 5).
        """
        super().__init__()
        self.hash_grid = hash_grid
        self.raw_volume = raw_volume
        self.n = n
        
        # Structural Encoder: 2 hidden layers, 32 channels
        self.structural_encoder = nn.Sequential(
            nn.Linear(n**3, 32), nn.ReLU(),
            nn.Linear(32, 32), nn.ReLU()
        )
        
        # FiLM Linear Layer: Maps structural representation (32) to scale and shift (128)
        self.film_linear = nn.Linear(32, 128)
        
        # Main MLP: 4 hidden layers, 64 channels
        self.main_mlp = nn.Sequential(
            nn.Linear(64, 64), nn.ReLU(),
            nn.Linear(64, 64), nn.ReLU(),
            nn.Linear(64, 64), nn.ReLU(),
            nn.Linear(64, 64), nn.ReLU()
        )
        
        # Output head for multi-task learning (intensity, gradient, local mean, local std)
        self.output_head = nn.Linear(64, 4)

    def get_local_patch(self, coords):
        """
        Sample a local patch of n^3 voxels centered at coords.
        coords: Tensor of shape (N, 3) with integer voxel coordinates.
        Returns: Tensor of shape (N, n^3)
        """
        N = coords.shape[0]
        half = self.n // 2
        offsets = torch.arange(-half, half + 1)
        px, py, pz = torch.meshgrid(offsets, offsets, offsets, indexing='ij')
        patch_coords = torch.stack([px, py, pz], dim=-1).reshape(-1, 3) # (n^3, 3)
        
        coords_exp = coords.unsqueeze(1) # (N, 1, 3)
        patch_coords_exp = patch_coords.unsqueeze(0) # (1, n^3, 3)
        sample_coords = coords_exp + patch_coords_exp # (N, n^3, 3)
        
        # Clamp to volume bounds
        sample_coords = torch.clamp(sample_coords, 0, torch.tensor(self.raw_volume.shape) - 1)
        
        # Flatten indices for gather
        D, H, W = self.raw_volume.shape
        idx = sample_coords[..., 0] * H * W + sample_coords[..., 1] * W + sample_coords[..., 2]
        idx = idx.long()
        
        patch_values = self.raw_volume.view(-1)[idx] # (N, n^3)
        return patch_values

    def forward(self, coords):
        """
        Args:
            coords: Tensor of shape (N, 3) with 3D coordinates.
        Returns:
            features: Tensor of shape (N, 64) from the final hidden layer.
            predictions: Tensor of shape (N, 4) for intensity, gradient, mean, std.
        """
        # Pathway 1: Positional Encoding
        F_pos = self.hash_grid(coords) # (N, 64)
        
        # Pathway 2: Structural Encoding
        patches = self.get_local_patch(coords) # (N, n^3)
        F_struct = self.structural_encoder(patches) # (N, 32)
        
        # FiLM Modulation
        film_params = self.film_linear(F_struct) # (N, 128)
        gamma = film_params[:, :64] # (N, 64)
        beta = film_params[:, 64:] # (N, 64)
        F_mod = F_pos * (gamma + 1.0) + beta # (N, 64)
        
        # Main MLP with Residual Connection
        mlp_out = self.main_mlp(F_mod) # (N, 64)
        x = F_mod + mlp_out # (N, 64)
        
        # Final hidden layer activations
        features = x
        
        # Predictions
        predictions = self.output_head(features) # (N, 4)
        
        return features, predictions
```

## Subtopics

### [FA-INR](10479-fa-inr.md)
FA-INR is a structure-aware primitive learning method for volume exploration that replaces the fixed spatial encoding of Explorable INR with a learnable cross-attention mechanism and Mixture-of-Experts (MoE) routing. It solves the problem of adapting neural representations to complex, spatially varying patterns in scientific simulation data by concentrating model capacity where it is needed most, while maintaining the ability to query continuous parameter spaces. Unlike its parent Explorable INR, FA-INR does not propagate uncertainty estimates, limiting its use in safety-critical workflows.

### Implicit Neural Representation (INR) for 2D-3D Registration
A dataset-free, patient-specific 2D-3D registration framework based on an implicit neural representation (INR) network and a composite similarity measure, used for non-coplanar radiation therapy with limited-angle CBCT scans [S2, abstract only, unofficial]. It uses a lightweight three-layer MLP followed by average pooling to calculate rigid motion parameters, transforms the 3D volume, generates Digitally Reconstructed Radiographs (DRRs) via Radon transform, and uses a composite similarity measure (pixel-wise intensity and gradient differences) to reduce domain gap impact [S2, abstract only, unofficial]. In simulation data experiments with 45-degree discrepancy, it achieved sub-millimeter accuracy in translations and sub-degree accuracy in rotations [S2, abstract only, unofficial]. In phantom experiments, translation errors were <2mm and sub-degree accuracy was achieved for pitch and roll [S2, abstract only, unofficial].

## Operating characteristics
*   **Time Complexity:** Not given in the sources [S1, unofficial].
*   **Memory Complexity:** Not given in the sources, though high-resolution volumes may exceed memory limits for comparison methods, requiring high-RAM servers [S1, unofficial].
*   **Measured Performance:** Not given in the sources for the Volume Exploration-Optimized INR itself [S1, unofficial].
*   **Scaling:** Not given in the sources.
*   **Strengths:** Captures local structural context and spatial coherence; jointly estimates intensity, gradient, and local statistics; uses FiLM for effective feature combination; residual connections stabilize learning [S1, unofficial].
*   **Limitations:** Relies on a fixed patch size (n=5); training is limited to 100 epochs; downstream tasks require thresholding (e.g., 0.5) [S1, unofficial].
*   **When to use:** Volume exploration tasks, specifically image-based transfer function design, where local structural context is important [S1, unofficial].
*   **When not to use:** Safety-critical workflows requiring uncertainty estimates (use FA-INR's parent or other methods if uncertainty is needed, though FA-INR also lacks this) [S1, unofficial].

## Background
The Volume Exploration-Optimized INR, also known as NeuVolEx, was introduced in the paper "NeuVolEx: Implicit Neural Features for Volume Exploration" [S1, unofficial]. It was developed to address the limitations of previous INR approaches that focused primarily on outputs rather than learned feature representations [S1, unofficial]. The method uses a dual-pathway design to process positional and structural inputs, combining them via FiLM modulation [S1, unofficial]. It is used for volume exploration tasks, particularly image-based transfer function design, allowing users to assign optical properties based on voxel confidence in ROI classes [S1, unofficial].

## Sources
[S1] NeuVolEx: Implicit Neural Features for Volume Exploration, https://doi.org/10.48550/arxiv.2604.11172, fetched 2026-10-26 (unofficial)
[S2] Fast volume rendering using a shear-warp factorisation ... - ResearchGate, https://www.researchgate.net/publication/243763776_Fast_volume_rendering_using_a_shear-warp_factorisation_of_the_viewing_transform, fetched 2026-10-27 (abstract only, unofficial)
