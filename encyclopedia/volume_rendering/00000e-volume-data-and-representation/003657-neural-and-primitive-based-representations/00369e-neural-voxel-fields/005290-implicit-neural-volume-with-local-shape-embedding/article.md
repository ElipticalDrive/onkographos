# Implicit Neural Volume with Local Shape Embedding

Also called: Local Shape Embedding, Neural Volume Representation

## Overview
Implicit Neural Volume with Local Shape Embedding is a 3D reconstruction method that represents geometry as a grid of latent codes, where each voxel stores a local shape embedding rather than a direct scalar field value [S1]. It solves the problem of efficiently encoding local geometric details from sparse depth inputs by using a learned encoder to map local point clouds to latent vectors and a decoder to predict Signed Distance Function (SDF) values [S1]. Unlike standard Neural Voxel Fields that may use fixed or simple features, this method explicitly learns a data-driven embedding of local shapes via an AutoEncoder-like architecture, allowing for compact representation of complex surfaces [S1].

## How it works
**Inputs and Outputs:**
*   **Input:** A sequence of depth maps $D_t$ with known intrinsic camera parameters.
*   **Output:** A global implicit neural volume $V_w$ consisting of voxels, each containing a 3D position $p$ and a latent code $l$, along with a weight volume $W_{csn}$. The volume allows querying the SDF $s$ for any 3D point $x$.

**Data Structures:**
1.  **Voxel ($v$):** A structure containing:
    *   $p$: 3D position in space (float3).
    *   $l$: Latent code representing local geometry (float32 vector of dimension 8) [S1].
2.  **Weight Volume ($W_{csn}$):** A grid parallel to the voxel grid where each cell stores an integer value representing the number of 3D points associated with that voxel [S1].
3.  **Voxel Resolution:** Fixed at 2 cm. Each latent code represents the geometry of a $2\text{cm}^3$ volume around its position [S1].

**Network Architectures:**
1.  **Depth Encoder ($E$):** Modified from PointNet [S1].
    *   Input: Local point cloud $p$ (points with normals).
    *   Architecture: 4 fully connected layers with sizes $[128, 128, 128, 8]$.
    *   Activation: ReLU on the first three layers; no activation on the last layer [S1].
    *   Aggregation: Average pooling layer aggregates information within the local region [S1].
    *   Output: Latent code $l = E(p)$ [S1].
2.  **Shape Decoder ($D$):** Multi-Layer Perceptron (MLP).
    *   Input: Latent code $l_i$ and local coordinate $\bar{x}_i$.
    *   Architecture: 4 fully connected layers with sizes $[128, 128, 128, 1]$ [S1].
    *   Output: SDF value [S1].

**Algorithm Steps:**

1.  **Unprojection:** At frame $t$, unproject the depth map $D_t$ to a 3D point cloud using intrinsic parameters [S1].
2.  **Segmentation:** Segment the 3D point cloud into overlapping local point clouds $\{p_0, p_1, ..., p_n\}$, each bounded by 3D cuboids [S1].
3.  **Encoding:** For each local point cloud $p_i$, compute the latent code $l_i = E(p_i)$ using the Depth Encoder [S1].
4.  **Volume Initialization (Single-View):**
    *   Initialize the single-view implicit volume $V_{cst}$ and weight volume $W_{cstn}$.
    *   For each voxel $v$ in the grid, if it contains points from the local point clouds, update its latent code $l$ and weight $w$ based on the encoded features. Specifically, $W_{cstn}$ values are set to the number of 3D points associated with the voxel [S1].
5.  **Bi-level Fusion (Global Update):**
    *   **Local-level Fusion:** Integrate latent codes from the single-view volume $V_{cst}$ into the global volume $V_w$ using weighted averaging at the local level [S1].
    *   **Global-level Fusion:** Optimize the global volume $V_w$ via neural rendering to ensure global consistency. Sample pixels, unproject to rays, and optimize latent codes using the Adam optimizer [S1].
6.  **SDF Query (Inference/Rendering):** To query the SDF $s$ for a 3D point $x$:
    *   Retrieve the 8 neighboring voxels $v_0...v_7$ in the volume grid [S1].
    *   For each neighbor $i$, compute the local coordinate: $\bar{x}_i = x - p_i$ [S1].
    *   Compute the SDF using the weighted sum of decoder outputs:
        $$ s = \frac{\sum_{i=0}^{7} w(\bar{x}_i, p_i) \mathcal{D}(l_i, \bar{x}_i)}{\sum_{i=0}^{7} w(\bar{x}_i, p_i)} $$
        where $w(\cdot, \cdot)$ is the weight of trilinear interpolation based on the position of $x$ relative to the voxel centers, and $\mathcal{D}$ is the Shape Decoder [S1].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import torch
import torch.nn as nn
import math

class DepthEncoder(nn.Module):
    """Modified PointNet encoder for local shape embedding."""
    def __init__(self):
        super(DepthEncoder, self).__init__()
        # 4 fully connected layers: [128, 128, 128, 8]
        self.fc1 = nn.Linear(3, 128) # Assuming input is 3D points (x,y,z)
        self.fc2 = nn.Linear(128, 128)
        self.fc3 = nn.Linear(128, 128)
        self.fc4 = nn.Linear(128, 8)
        self.relu = nn.ReLU()

    def forward(self, points):
        # points: (N, 3)
        x = self.relu(self.fc1(points))
        x = self.relu(self.fc2(x))
        x = self.relu(self.fc3(x))
        x = self.fc4(x) # No activation on last layer
        
        # Aggregate using average pooling
        # x shape: (N, 8) -> (1, 8)
        latent_code = torch.mean(x, dim=0, keepdim=True)
        return latent_code

class ShapeDecoder(nn.Module):
    """MLP decoder for SDF prediction."""
    def __init__(self):
        super(ShapeDecoder, self).__init__()
        # 4 fully connected layers: [128, 128, 128, 1]
        # Input is concatenation of latent code (8) and local coord (3) -> 11
        self.fc1 = nn.Linear(11, 128)
        self.fc2 = nn.Linear(128, 128)
        self.fc3 = nn.Linear(128, 128)
        self.fc4 = nn.Linear(128, 1)

    def forward(self, latent_code, local_coord):
        # latent_code: (B, 8), local_coord: (B, 3)
        x = torch.cat([latent_code, local_coord], dim=1)
        x = torch.relu(self.fc1(x))
        x = torch.relu(self.fc2(x))
        x = torch.relu(self.fc3(x))
        sdf = self.fc4(x)
        return sdf

def trilinear_weight(x, p_i, voxel_size=0.02):
    """
    Computes the trilinear interpolation weight for point x relative to voxel center p_i.
    Assumes x is in local coordinates relative to the voxel grid.
    Returns a scalar weight.
    Note: Standard trilinear interpolation involves 8 neighbors. 
    Here, w(x_bar_i, p_i) represents the influence of voxel i on point x.
    For implementation, this usually refers to the basis function value.
    """
    # Calculate distance in voxel units
    dist = (x - p_i) / voxel_size
    # Clamp to [-1, 1] for valid interpolation range within the voxel
    dist = torch.clamp(dist, -1.0, 1.0)
    # Simple linear weight for demonstration; actual trilinear uses product of linear weights
    # w = (1 - |dist|) for 1D linear. For 3D trilinear:
    # w = (1 - |dx|) * (1 - |dy|) * (1 - |dz|)
    # However, the source defines w as "weight of trilinear interpolation".
    # In the context of the formula s = sum(w * D) / sum(w), w acts as the interpolation kernel.
    # We return the kernel value.
    w = (1 - torch.abs(dist[:, 0])) * (1 - torch.abs(dist[:, 1])) * (1 - torch.abs(dist[:, 2]))
    return w

def query_sdf(x, voxel_grid, encoder, decoder):
    """
    Query SDF for point x from the implicit neural volume.
    voxel_grid: List of voxels, each is {'p': (3,), 'l': (8,)}
    """
    # Retrieve 8 neighboring voxels
    # Implementation detail: Find the grid cell containing x, then identify 8 neighbors
    # This requires a spatial index (e.g., hash map or grid array).
    # For the sake of the algorithm description, we assume get_8_neighbors(x) returns indices.
    neighbor_indices = get_8_neighbors(x, voxel_grid) # [G]: Helper to find neighbors in grid
    
    numerator = 0.0
    denominator = 0.0
    
    for idx in neighbor_indices:
        v = voxel_grid[idx]
        p_i = v['p']
        l_i = v['l']
        
        # Transform to local coordinate
        x_bar_i = x - p_i
        
        # Compute trilinear weight
        w = trilinear_weight(x_bar_i, torch.zeros(3), voxel_size=0.02) # [G]: p_i is origin for local coord, so dist is x_bar_i
        
        # Decode SDF
        sdf_val = decoder(l_i, x_bar_i)
        
        numerator += w * sdf_val
        denominator += w
        
    if denominator == 0:
        return 0.0 # [G]: Fallback for out-of-bounds
        
    return numerator / denominator
```

**Parameters:**
*   Voxel resolution: 2 cm [S1].
*   Latent code dimension: 8 [S1].
*   Encoder layers: $[128, 128, 128, 8]$ with ReLU [S1].
*   Decoder layers: $[128, 128, 128, 1]$ [S1].
*   Training points per CAD model: 2000 seed points for encoder, 1000 for decoder [S1].
*   Global optimization: Adam optimizer, 5 iterations per image, 5000 pixels sampled per image, 5 points/meter (coarse), 20 points/meter (fine) [S1].

**Edge Cases and Pitfalls:**
*   **Out-of-bounds:** If a query point $x$ has no neighboring voxels (e.g., outside the defined grid), the denominator in the SDF formula becomes zero. A fallback value (e.g., 0 or infinity) must be defined [G].
*   **Precision:** The use of float32 for latent codes and positions is standard, but small voxel sizes (2 cm) relative to scene scale may require careful coordinate normalization to avoid precision loss in the MLP [G].
*   **Degenerate Input:** If a local point cloud $p_i$ is empty, the encoder cannot produce a valid latent code. The segmentation step must ensure non-empty local regions [G].

## Operating characteristics
*   **Time Complexity:** Encoding is $O(N)$ for $N$ points in a local cloud. Decoding is $O(1)$ per query (constant number of neighbors). Global optimization involves iterative neural rendering, which is expensive per frame but allows incremental updates [S1].
*   **Memory Complexity:** Proportional to the number of voxels in the grid. Each voxel stores a fixed-size latent code (8 floats) and a position (3 floats) [S1].
*   **Strengths:** Compact representation due to latent codes; effective at capturing local geometric details; supports incremental updates via bi-level fusion [S1].
*   **Limitations:** Requires careful tuning of voxel resolution; performance depends on the quality of the depth encoder; global consistency requires iterative optimization [S1].
*   **When to use:** Suitable for dense 3D reconstruction from RGB-D sequences where memory efficiency and local detail preservation are critical [S1].
*   **When not to use:** Not ideal for real-time applications requiring instantaneous SDF queries without pre-computation, or for scenes with extremely large scales where 2 cm voxels are insufficient [S1].

## Background
The method was introduced in "BNV-Fusion: Dense 3D Reconstruction using Bi-level Neural Volume Fusion" [S1]. It builds on the concept of Neural Voxel Fields, extending them with a learned local shape embedding to improve geometric accuracy [S1]. The encoder is adapted from PointNet, a pioneering network for point cloud processing [S1]. The method was evaluated on datasets such as 3D Scene Dataset, ICL-NUIM, and ScanNet [S1].

## Sources
[S1] BNV-Fusion: Dense 3D Reconstruction using Bi-level Neural Volume Fusion, https://doi.org/10.1109/CVPR52688.2022.00607, fetched 2026-10-30
