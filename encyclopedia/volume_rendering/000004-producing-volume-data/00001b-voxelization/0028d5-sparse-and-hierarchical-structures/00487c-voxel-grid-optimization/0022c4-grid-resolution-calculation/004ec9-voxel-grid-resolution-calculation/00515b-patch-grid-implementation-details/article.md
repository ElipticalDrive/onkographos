# Patch-Grid Implementation Details

Also called: Feature volume sampling, Patch volume resolution

## Overview
Patch-Grid decomposes a 3D shape into local surface patches, each represented by a learnable feature volume and a shared decoder to accurately capture fine geometric details and thin structures in CAD models [S1]. It differs from global implicit fields by using adaptive, patch-specific resolutions based on local shape diameter, and explicitly optimizes per-patch feature volumes with merge constraints to maintain continuity [S1].

## How it works

### Inputs and Outputs
**Inputs:**
*   A set of segmented surface patches from a 3D shape [S1].
*   Ground truth surface points and normals for each patch [S1].
*   Spatial points within the bounding volume of each patch [S1].

**Outputs:**
*   A set of patch feature volumes (latent codes) [S1].
*   A trained decoder (MLP) [S1].
*   A merge grid (octree) defining spatial relationships between patches [S1].

### Data Structures

**Patch Feature Volume ($V_p$):**
*   **Layout:** A 3D grid of latent codes [S1].
*   **Dimensions:** Resolution varies per patch, ranging from $2^3$ to $2^6$ [S1].
*   **Content:** Each cell contains a 256-dimensional latent vector [S1].
*   **Resolution Calculation:** The grid cell size must not exceed 2.5 times the average shape diameter of the patch, computed using LIBIGL [S1].

**Decoder:**
*   **Structure:** 1-layer Multi-Layer Perceptron (MLP) [S1].
*   **Hidden Layer:** 128 neurons [S1].
*   **Activation:** SoftPlus with $\beta=100$ for the hidden layer [S1]. No activation function for the output layer [S1].
*   **Input:** Concatenation of position coordinates and the latent code sampled from the patch feature volume at the corresponding spatial location [S1].
*   **Output:** Signed Distance Function (SDF) value and surface normal [S1].

**Merge Grid ($G$):**
*   **Structure:** Octree [S1].
*   **Max Depth:** Empirically set to 8 [S1].
*   **Purpose:** Defines merge constraints between adjacent patch feature volumes to ensure continuity [S1].

### Sampling Strategy

1.  **Sampling Pool Construction:** For each patch, uniformly sample 1,000 surface points and 1,000 spatial points within each cell of the corresponding feature volume to form the sampling pool [S1].
2.  **Training Iteration Sampling:** In each training iteration, randomly sample 10,000 surface points and 10,000 spatial points from the prepared sampling pool [S1].

### Training Procedures

The method supports three training modes, all using the ADAM optimizer with default hyperparameters [S1].

**1. Pretraining the Decoder:**
*   **Dataset:** 1,849 patches from 25 CAD shapes [S1].
*   **Batch Size:** 240,000 [S1].
*   **Iterations:** 200,000 [S1].
*   **Learning Rate:** Initial 0.001, decayed by factor 0.3 at iterations 150,000 and 175,000 [S1].
*   **Objective:** Patch fitting loss ($L_{patch}$) only; the merging loss ($L_{merge}$) is excluded [S1].
*   **Duration:** Approximately 20 hours on NVIDIA RTX4090 [S1].

**2. Shape Fitting (Training from Scratch):**
*   **Targets:** Both the decoder and patch feature volumes are updated [S1].
*   **Iterations:** 500 [S1].
*   **Learning Rate:** Initial 0.001, decayed by factor 0.3 at iterations 470 and 485 [S1].

**3. Shape Fitting (With Pretrained Decoder):**
*   **Targets:** Only patch feature volumes are updated; the decoder is fixed [S1].
*   **Iterations:** 300 [S1].
*   **Learning Rate:** Initial 0.001, decayed by factor 0.3 at iterations 270 and 285 [S1].

**4. Test-Time Optimization (Shape Updating):**
*   **Context:** Used when editing a 3D shape, updating only the changed patches and their affected neighbors [S1].
*   **Targets:** Only changed patch feature volumes are updated [S1].
*   **Iterations:** 80 [S1].
*   **Learning Rate:** Initial 0.001, decayed by factor 0.3 at iterations 65 and 72 [S1].

### Loss Function

The total loss is a weighted sum of five terms [S1]. The balance weights are:
*   $\lambda_{surface} = 200$
*   $\lambda_{normal} = 50$
*   $\lambda_{SDF} = 50$
*   $\lambda_{eikonal} = 5$
*   $\lambda_{merge} = 400$

The loss terms are:
*   $L_{surface}$: Surface fitting loss [S1].
*   $L_{normal}$: Normal consistency loss [S1].
*   $L_{SDF}$: SDF regression loss [S1].
*   $L_{eikonal}$: Eikonal equation regularization [S1].
*   $L_{merge}$: Merge constraint loss to ensure continuity between adjacent patches [S1].

### Pseudocode

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import torch
import torch.nn as nn
import torch.optim as optim

def softplus(x, beta=100):
    return torch.nn.functional.softplus(x, beta=beta)

class Decoder(nn.Module):
    def __init__(self):
        super().__init__()
        self.hidden = nn.Linear(259, 128)
        self.output = nn.Linear(128, 4)

    def forward(self, pos, latent_code):
        x = torch.cat([pos, latent_code], dim=-1)
        x = softplus(self.hidden(x))
        out = self.output(x)
        return out[:, 0], out[:, 1:]

def compute_loss(sdf, normal, pos, surface_points, surface_normals, spatial_points, merge_constraints):
    l_surface = torch.mean(torch.abs(sdf))
    normal_unit = normal / (torch.norm(normal, dim=-1, keepdim=True) + 1e-8)
    l_normal = torch.mean(1.0 - torch.sum(normal_unit * surface_normals, dim=-1))
    l_sdf = torch.mean(sdf ** 2)
    grad_sdf = torch.autograd.grad(sdf.sum(), pos, create_graph=True)[0]
    l_eikonal = torch.mean((torch.norm(grad_sdf, dim=-1, keepdim=True) - 1.0) ** 2)
    l_merge = torch.mean(merge_constraints) if merge_constraints.numel() > 0 else torch.tensor(0.0)
    return (200 * l_surface + 50 * l_normal + 50 * l_sdf + 5 * l_eikonal + 400 * l_merge)

def train_patch_grid(patches, decoder, feature_volumes, optimizer, num_iterations, lr, decay_steps, decay_factor):
    decoder.train()
    for step in range(num_iterations):
        if step in decay_steps:
            for param_group in optimizer.param_groups:
                param_group['lr'] *= decay_factor

        surface_points, surface_normals, spatial_points = sample_from_pool(patches)
        latent_codes = trilinear_sample(feature_volumes, spatial_points)
        sdf, normal = decoder(spatial_points, latent_codes)
        merge_constraints = compute_merge_constraints(feature_volumes, spatial_points)
        loss = compute_loss(sdf, normal, spatial_points, surface_points, surface_normals, spatial_points, merge_constraints)
        optimizer.zero_grad()
        loss.backward()
        optimizer.step()
    return decoder, feature_volumes

def sample_from_pool(patches):
    all_surface_points = torch.cat([p.surface_pool for p in patches])
    all_surface_normals = torch.cat([p.normal_pool for p in patches])
    all_spatial_points = torch.cat([p.spatial_pool for p in patches])
    idx = torch.randperm(len(all_surface_points))[:10000]
    return all_surface_points[idx], all_surface_normals[idx], all_spatial_points[idx]

def trilinear_sample(grid, points):
    # Standard trilinear interpolation from a 3D grid at given coordinates
    # Not given in the sources.
    return torch.zeros(points.shape[0], 256)

def compute_merge_constraints(feature_volumes, spatial_points):
    # Derive merge constraints from the merge grid cells accordingly
    # Not given in the sources.
    return torch.tensor([])
```

### Edge Cases and Pitfalls
*   **Uniform Resolution:** Using uniform patch volume sizes results in coarse resolution that fails to accurately fit thin tubes and slender geometry [S1].
*   **Missing Surface Loss:** Removing $L_{surface}$ leads to large fitting errors and offset from the ground truth surface, even if other losses are present [S1].
*   **Missing Merge Loss:** Removing $L_{merge}$ leads to artifacts due to lack of proper CSG operations between patches [S1].
*   **Missing Normal Loss:** Removing $L_{normal}$ leads to severe artifacts [S1].

## Subtopics

### Adaptive Patch Feature Volume
Determines the resolution of each patch's feature volume adaptively based on the local shape diameter to accurately capture slender geometry [S1]. It calculates the shape diameter for each patch and assigns volume resolution proportional to the diameter; if the diameter is small, resolution is increased, otherwise, a coarser resolution is used [S1]. This avoids the pitfall of uniform patch volume sizes, which fail to fit thin tubes accurately [S1]. Adaptive resolution successfully resolves thin tubes, whereas uniform resolution does not [S1].

## Operating characteristics

*   **Hardware:** Experiments conducted on NVIDIA RTX4090 GPU and Intel i9 13900kf CPU [S1].
*   **Performance:** Pretraining the decoder takes approximately 20 hours for 200k iterations [S1].
*   **Memory:** Batch size of 240k is used to fully utilize GPU memory during pretraining [S1].
*   **Scalability:** The method scales with the number of patches and their resolutions [S1].
*   **Strengths:** Excellent accuracy for thin structures, sharp boundaries, and clean boundary curves [S1].
*   **Limitations:** Requires segmentation of the shape into patches [S1].

## Background

Patch-Grid was introduced in the paper "Patch-Grid: An Efficient and Feature-Preserving Neural Implicit Surface Representation" [S1]. It builds on the idea of decomposing complex shapes into manageable local representations [S1]. The method uses LIBIGL for computing average shape diameter [S1] and follows training strategies similar to Park et al. [2019] for test-time optimization [S1]. The SoftPlus activation function with $\beta=100$ is suggested by Atzmon and Lipman [2020] [S1].

## Sources

[S1] Patch-Grid: An Efficient and Feature-Preserving Neural Implicit Surface Representation, https://doi.org/10.1145/3727142, fetched 2026-10-28
