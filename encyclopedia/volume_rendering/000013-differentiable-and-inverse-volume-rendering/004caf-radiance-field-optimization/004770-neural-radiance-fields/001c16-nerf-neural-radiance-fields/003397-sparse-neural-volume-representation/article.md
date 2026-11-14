# Sparse Neural Volume Representation

Also called: Sparse Neural Volume Radiance Field, Sparse Volume for Radiance Fields

## Overview
Sparse Neural Volume Representation models a radiance field using a sparse 3D grid of per-voxel neural features that approximately cover the scene surface, enabling scalable reconstruction for large scenes [S1]. Unlike standard NeRF which relies on a global MLP without intermediate spatial structure, this method uses explicit voxel features to improve scalability for large scenes and generalization across scenes [S1].

## How it works
**Inputs:** A sequence of RGB images with known camera poses (obtained via SLAM or SfM) [S1].
**Outputs:** A sparse neural volume $V$ containing per-voxel features, and a shared MLP decoder $R$ [S1].

**Data Structures:**
*   **Sparse Neural Volume ($V$):** A 3D grid structure where only voxels covering the scene surface contain data [S1]. It consists of two components:
    *   **Local Volume ($V_t$):** Reconstructed for each input frame $t$ in the canonical world space [S1].
    *   **Global Volume ($V_g$):** Reconstructed for the entire sequence in the canonical world space [S1].
*   **MLP Decoder ($R$):** A neural network shared across all volumes and scenes [S1].

**Evaluation Steps:**
To evaluate the radiance field at a 3D location $x$ for a viewing direction $d$:
1.  **Tri-linear Interpolation:** Sample a feature vector $\mathcal{V}(x)$ from the sparse volume $V$ at location $x$ using tri-linear interpolation [S1].
2.  **MLP Regression:** Input the feature vector $\mathcal{V}(x)$, the 3D location $x$, and the viewing direction $d$ into the MLP network $R$ [S1].
3.  **Output:** The network $R$ regresses the volume density $\sigma$ and view-dependent radiance $c$ [S1].

**Formula:**
The relationship is defined as:
$$ \sigma, c = R(x, d, \mathcal{V}(x)) $$
Where:
*   $\sigma$: Volume density.
*   $c$: View-dependent radiance.
*   $x$: 3D location in the canonical world space.
*   $d$: Viewing direction.
*   $R$: The MLP decoder network.
*   $\mathcal{V}(x)$: The trilinearly interpolated feature vector from the sparse volume at $x$ [S1].

**Parameters:** Not given in the sources. [S1]

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import math
import numpy as np

def evaluate_radiance_field(x, d, sparse_volume, mlp_decoder):
    """
    Evaluates density and radiance at point x for direction d.
    
    Args:
        x: 3D location (float32, shape [3])
        d: Viewing direction (float32, shape [3])
        sparse_volume: Sparse 3D grid/dict containing per-voxel features
        mlp_decoder: MLP network R
        
    Returns:
        sigma: Volume density (float32)
        c: View-dependent radiance (float32, shape [3])
    """
    # Step 1: Tri-linearly sample feature vector from sparse volume
    ix, iy, iz = x
    x0, y0, z0 = math.floor(ix), math.floor(iy), math.floor(iz)
    x1, y1, z1 = x0 + 1, y0 + 1, z0 + 1
    
    # Fetch 8 corner features (default to zero vector if voxel is empty)
    f000 = sparse_volume.get((x0, y0, z0), np.zeros(3))
    f100 = sparse_volume.get((x1, y0, z0), np.zeros(3))
    f010 = sparse_volume.get((x0, y1, z0), np.zeros(3))
    f110 = sparse_volume.get((x1, y1, z0), np.zeros(3))
    f001 = sparse_volume.get((x0, y0, z1), np.zeros(3))
    f101 = sparse_volume.get((x1, y0, z1), np.zeros(3))
    f011 = sparse_volume.get((x0, y1, z1), np.zeros(3))
    f111 = sparse_volume.get((x1, y1, z1), np.zeros(3))
    
    dx, dy, dz = ix - x0, iy - y0, iz - z0
    
    # Tri-linear interpolation
    c00 = f000 * (1 - dx) + f100 * dx
    c10 = f010 * (1 - dx) + f110 * dx
    c01 = f001 * (1 - dx) + f101 * dx
    c11 = f011 * (1 - dx) + f111 * dx
    
    c0 = c00 * (1 - dy) + c10 * dy
    c1 = c01 * (1 - dy) + c11 * dy
    
    feature_vector = c0 * (1 - dz) + c1 * dz
    
    # Step 2: Pass feature vector, location, and direction into MLP
    density_and_radiance = mlp_decoder(x, d, feature_vector)
    
    # Step 3: Extract density and radiance
    sigma = density_and_radiance.density
    c = density_and_radiance.radiance
    
    return sigma, c
```

**Edge Cases and Pitfalls:**
*   **Empty Voxels:** Since the volume is sparse, querying a location outside the covered surface region may return undefined or zero features [S1]. The method assumes $x$ lies within the approximate surface coverage [S1].

## Subtopics

### Local Volume Reconstruction
Regresses a local neural volume $V_t$ for each input frame using the current image and neighboring views [S1].

### Global Volume Fusion
Fuses local volumes into a global volume $V_g$ for the entire sequence, inspired by TSDF fusion workflows [S1].

### Cross-Scene Generalization
Leverages neural networks trained across multiple scenes to predict neural volumes, bypassing per-scene optimization [S1].

## Operating characteristics
*   **Scalability:** Improves scalability for large scenes compared to global MLP-based NeRF by using a sparse volume grid [S1].
*   **Speed:** Enables fast radiance field reconstruction through direct network inference, bypassing slow per-scene optimization [S1].
*   **Quality:** Can be fine-tuned to boost quality, but initial reconstruction is achieved via inference [S1].
*   **Comparison:** Unlike Neural Sparse Voxel Fields which relies on per-scene optimization, this method generalizes across scenes [S1].
*   **Time and Memory Complexity:** Not given in the sources. [S1]

## Background
The method was introduced in "NeRFusion: Fusing Radiance Fields for Large-Scale Scene Reconstruction" [S1]. It builds on the classical TSDF (Truncated Signed Distance Function) fusion workflow, starting from per-view geometry and fusing reconstructions across key frames [S1]. It addresses the limitations of previous methods that required slow per-scene optimization by proposing a deep neural network trained across scenes [S1].

## Sources
[S1] NeRFusion: Fusing Radiance Fields for Large-Scale Scene Reconstruction, https://doi.org/10.1109/cvpr52688.2022.00537, fetched 2026-10-26
