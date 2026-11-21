# Patch-Grid Implementation Details

Also called: Feature volume sampling, Patch volume resolution

## Overview
Patch-Grid is a neural implicit surface representation that decomposes a 3D shape into local surface patches, each represented by a learnable feature volume and a shared decoder. It solves the problem of accurately capturing fine geometric details and thin structures in complex CAD models, which global implicit fields often fail to represent due to resolution limits or smoothing artifacts. It differs from global voxel grids by using adaptive, patch-specific resolutions based on local shape diameter, and from other sparse structures by explicitly optimizing feature volumes per patch with specific loss constraints for merging.

## How it works

### Inputs and Outputs
**Inputs:**
*   A set of segmented surface patches from a 3D shape.
*   Ground truth surface points and normals for each patch.
*   Spatial points within the bounding volume of each patch.

**Outputs:**
*   A set of patch feature volumes (latent codes).
*   A trained decoder (MLP).
*   A merge grid (octree) defining spatial relationships between patches.

### Data Structures

**Patch Feature Volume ($V_p$):**
*   **Layout:** A 3D grid of latent codes.
*   **Dimensions:** Resolution varies per patch, ranging from $2^3$ to $2^6$.
*   **Content:** Each cell contains a 256-dimensional latent vector.
*   **Resolution Calculation:** The grid cell size must not exceed 2.5 times the average shape diameter of the patch, computed using LIBIGL [S1].

**Decoder:**
*   **Structure:** 1-layer Multi-Layer Perceptron (MLP).
*   **Hidden Layer:** 128 neurons.
*   **Activation:** SoftPlus with $\beta=100$ for the hidden layer. No activation function for the output layer.
*   **Input:** Concatenation of position coordinates and the latent code sampled from the patch feature volume at the corresponding spatial location.
*   **Output:** Signed Distance Function (SDF) value and surface normal.

**Merge Grid ($G$):**
*   **Structure:** Octree.
*   **Max Depth:** Empirically set to 8.
*   **Purpose:** Defines merge constraints between adjacent patch feature volumes to ensure continuity.

### Sampling Strategy

1.  **Sampling Pool Construction:** For each patch, uniformly sample 1,000 surface points and 1,000 spatial points within each cell of the corresponding feature volume. This forms the sampling pool for that patch [S1].
2.  **Training Iteration Sampling:** In each training iteration, randomly sample 10,000 surface points and 10,000 spatial points from the prepared sampling pool [S1].

### Training Procedures

The method supports three training modes. All use the ADAM optimizer with default hyperparameters [S1].

**1. Pretraining the Decoder:**
*   **Dataset:** 1,849 patches from 25 CAD shapes.
*   **Batch Size:** 240,000.
*   **Iterations:** 200,000.
*   **Learning Rate:** Initial 0.001. Decayed by factor 0.3 at iterations 150,000 and 175,000.
*   **Objective:** Patch fitting loss ($L_{patch}$) only. The merging loss ($L_{merge}$) is excluded.
*   **Duration:** Approximately 20 hours on NVIDIA RTX4090 [S1].

**2. Shape Fitting (Training from Scratch):**
*   **Targets:** Both the decoder and patch feature volumes are updated.
*   **Iterations:** 500.
*   **Learning Rate:** Initial 0.001. Decayed by factor 0.3 at iterations 470 and 485.

**3. Shape Fitting (With Pretrained Decoder):**
*   **Targets:** Only patch feature volumes are updated; the decoder is fixed.
*   **Iterations:** 300.
*   **Learning Rate:** Initial 0.001. Decayed by factor 0.3 at iterations 270 and 285.

**4. Test-Time Optimization (Shape Updating):**
*   **Context:** Used when editing a 3D shape, updating only the changed patches and their affected neighbors.
*   **Targets:** Only changed patch feature volumes are updated.
*   **Iterations:** 80.
*   **Learning Rate:** Initial 0.001. Decayed by factor 0.3 at iterations 65 and 72.

### Loss Function

The total loss is a weighted sum of five terms. The balance weights are:
*   $\lambda_{surface} = 200$
*   $\lambda_{normal} = 50$
*   $\lambda_{SDF} = 50$
*   $\lambda_{eikonal} = 5$
*   $\lambda_{merge} = 400$

The loss terms are:
*   $L_{surface}$: Surface fitting loss.
*   $L_{normal}$: Normal consistency loss.
*   $L_{SDF}$: SDF regression loss.
*   $L_{eikonal}$: Eikonal equation regularization.
*   $L_{merge}$: Merge constraint loss to ensure continuity between adjacent patches.

### Pseudocode

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import torch
import torch.nn as nn
import torch.optim as optim

# Standard SoftPlus activation
def softplus(x, beta=100):
    return torch.nn.functional.softplus(x, beta=beta)

class Decoder(nn.Module):
    def __init__(self):
        super(Decoder, self).__init__()
        # 1-layer MLP: Input dim depends on latent code size (256) + position (3)
        # Hidden layer: 128 neurons
        # Output: SDF (1) + Normal (3) = 4 values
        self.hidden = nn.Linear(259, 128)
        self.output = nn.Linear(128, 4)
        
    def forward(self, pos, latent_code):
        # Concatenate position and latent code
        x = torch.cat([pos, latent_code], dim=-1)
        x = softplus(self.hidden(x))
        out = self.output(x)
        sdf = out[:, 0]
        normal = out[:, 1:]
        return sdf, normal

def compute_loss(sdf, normal, pos, surface_points, surface_normals, 
                 spatial_points, merge_constraints, lambda_s=200, 
                 lambda_n=50, lambda_sdf=50, lambda_eik=5, lambda_m=400):
    # L_surface: Distance to surface points
    surface_dist = torch.abs(sdf)
    l_surface = torch.mean(surface_dist)
    
    # L_normal: Normal consistency
    # Assuming normal is normalized in decoder output or needs normalization
    normal_norm = torch.norm(normal, dim=-1, keepdim=True)
    normal_unit = normal / (normal_norm + 1e-8)
    # Dot product with ground truth normals
    l_normal = torch.mean(1.0 - torch.sum(normal_unit * surface_normals, dim=-1))
    
    # L_sdf: Regression to 0 at surface
    l_sdf = torch.mean(sdf ** 2)
    
    # L_eikonal: Gradient magnitude should be 1
    # Note: In practice, this requires gradient computation w.r.t pos
    # Simplified placeholder for gradient magnitude check
    grad_sdf = torch.autograd.grad(sdf.sum(), pos, create_graph=True)[0]
    grad_norm = torch.norm(grad_sdf, dim=-1, keepdim=True)
    l_eikonal = torch.mean((grad_norm - 1.0) ** 2)
    
    # L_merge: Constraint between adjacent patches
    # Placeholder for merge constraint logic based on merge grid G
    l_merge = 0.0 
    if merge_constraints is not None:
        # Example: enforce similar SDF values at boundaries
        l_merge = torch.mean(merge_constraints)
        
    total_loss = (lambda_s * l_surface + 
                  lambda_n * l_normal + 
                  lambda_sdf * l_sdf + 
                  lambda_eik * l_eikonal + 
                  lambda_m * l_merge)
    return total_loss

def train_patch_grid(patches, decoder, feature_volumes, optimizer, 
                     num_iterations, lr, decay_steps, decay_factor):
    decoder.train()
    for step in range(num_iterations):
        # Decay learning rate
        if step in decay_steps:
            for param_group in optimizer.param_groups:
                param_group['lr'] *= decay_factor
        
        # Sample points from pool
        surface_points, surface_normals, spatial_points = sample_from_pool(patches)
        
        # Forward pass
        # For each spatial point, sample latent code from feature volume
        latent_codes = sample_latent_codes(feature_volumes, spatial_points)
        sdf, normal = decoder(spatial_points, latent_codes)
        
        # Compute loss
        loss = compute_loss(sdf, normal, spatial_points, surface_points, 
                            surface_normals, spatial_points) # Merge constraints passed separately
        
        # Backward pass
        optimizer.zero_grad()
        loss.backward()
        optimizer.step()
        
    return decoder, feature_volumes

def sample_from_pool(patches):
    # Randomly sample 10,000 surface points and 10,000 spatial points
    # from the pre-computed sampling pool for each patch
    pass

def sample_latent_codes(feature_volumes, spatial_points):
    # Trilinear interpolation or nearest neighbor sampling from 3D grid
    pass
```

### Edge Cases and Pitfalls
*   **Uniform Resolution:** Using uniform patch volume sizes results in coarse resolution that fails to accurately fit thin tubes and slender geometry. Adaptive resolution is necessary [S1].
*   **Missing Surface Loss:** Removing $L_{surface}$ leads to large fitting errors and offset from the ground truth surface, even if other losses are present [S1].
*   **Missing Merge Loss:** Removing $L_{merge}$ leads to artifacts due to lack of proper CSG operations between patches [S1].
*   **Missing Normal Loss:** Removing $L_{normal}$ leads to severe artifacts [S1].

## Subtopics

### Adaptive Patch Feature Volume
Determines the resolution of each patch's feature volume adaptively based on the local shape diameter to accurately capture slender geometry. It calculates the shape diameter for each patch and assigns volume resolution proportional to the diameter; if the diameter is small, resolution is increased, otherwise, a coarser resolution is used. This avoids the pitfall of uniform patch volume sizes, which fail to fit thin tubes accurately. Adaptive resolution successfully resolves thin tubes, whereas uniform resolution does not [S1].

## Operating characteristics

*   **Hardware:** Experiments conducted on NVIDIA RTX4090 GPU and Intel i9 13900kf CPU [S1].
*   **Performance:** Pretraining the decoder takes approximately 20 hours for 200k iterations [S1].
*   **Memory:** Batch size of 240k is used to fully utilize GPU memory during pretraining [S1].
*   **Scalability:** The method scales with the number of patches and their resolutions. Adaptive resolution helps manage memory by using coarser grids for larger, simpler patches.
*   **Strengths:** Excellent accuracy for thin structures, sharp boundaries, and clean boundary curves. Preserves features better than global implicit fields.
*   **Limitations:** Requires segmentation of the shape into patches. Training time can be significant for complex shapes with many patches.

## Background

Patch-Grid was introduced in the paper "Patch-Grid: An Efficient and Feature-Preserving Neural Implicit Surface Representation" [S1]. It builds on the idea of decomposing complex shapes into manageable local representations. The method uses LIBIGL for computing average shape diameter [S1] and follows training strategies similar to Park et al. [2019] for test-time optimization. The SoftPlus activation function with $\beta=100$ is suggested by Atzmon and Lipman [2020] [S1].

## Sources

[S1] Patch-Grid: An Efficient and Feature-Preserving Neural Implicit Surface Representation, https://doi.org/10.1145/3727142, fetched 2026-10-28
