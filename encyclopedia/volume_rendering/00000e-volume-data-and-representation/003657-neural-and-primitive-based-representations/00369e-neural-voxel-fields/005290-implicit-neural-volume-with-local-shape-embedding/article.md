# Implicit Neural Volume with Local Shape Embedding

Also called: Local Shape Embedding, Neural Volume Representation

## Overview
Implicit Neural Volume with Local Shape Embedding represents 3D geometry as a grid of latent codes, where each voxel stores a local shape embedding rather than a direct scalar field value [S1]. It encodes local geometric details from single-view depth inputs by using a learned encoder to map local point clouds to latent vectors and a decoder to predict Signed Distance Function (SDF) values [S1]. Unlike standard Neural Voxel Fields that may use fixed or simple features, this method explicitly learns a data-driven embedding of local shapes via an AutoEncoder-like architecture, allowing for compact representation of complex surfaces [S1].

## How it works
**Inputs and Outputs:**
*   **Input:** Depth maps $D_t$ with known intrinsic camera parameters [S1].
*   **Output:** Global implicit neural volume $V_w$ consisting of voxels, each containing a 3D position $p$ and a latent code $l$, along with a weight volume $W_{csn}$ [S1]. The volume allows querying the SDF $s$ for any 3D point $x$ [S1].

**Data Structures:**
1.  **Voxel ($v$):** A structure containing:
    *   $p$: 3D position in space (float3).
    *   $l$: Latent code representing local geometry (float32 vector of dimension 8) [S1].
2.  **Weight Volume ($W_{csn}$):** A grid parallel to the voxel grid where each cell stores an integer value representing the number of 3D points associated with that voxel [S1].
3.  **Voxel Resolution:** Fixed at 2 cm. Each latent code represents the geometry of a $2\text{cm}^3$ volume around its position [S1].

**Network Architectures:**
1.  **Depth Encoder ($E$):** Modified from PointNet [S1].
    *   Input: Local point cloud $p$ (points with normals) [S1].
    *   Architecture: 4 fully connected layers with sizes $[128, 128, 128, 8]$ [S1].
    *   Activation: ReLU on the first three layers; no activation on the last layer [S1].
    *   Aggregation: Average pooling layer aggregates information within the local region [S1].
    *   Output: Latent code $l = E(p)$ [S1].
2.  **Shape Decoder ($D$):** Multi-Layer Perceptron (MLP) [S1].
    *   Input: Latent code $l_i$ and local coordinate $\bar{x}_i$ [S1].
    *   Architecture: 4 fully connected layers with sizes $[128, 128, 128, 1]$ [S1].
    *   Output: SDF value [S1].

**Algorithm Steps:**

1.  **Unprojection:** At frame $t$, unproject the depth map $D_t$ to a 3D point cloud using intrinsic parameters [S1].
2.  **Segmentation:** Segment the 3D point cloud into overlapping local point clouds $\{p_0, p_1, ..., p_n\}$, each bounded by 3D cuboids [S1].
3.  **Encoding:** For each local point cloud $p_i$, compute the latent code $l_i = E(p_i)$ using the Depth Encoder [S1].
4.  **Single-View Volume Construction:** Aggregate latent codes into a single-view implicit volume $V_{cst}$ [S1]. Initialize weight volume $W_{csn}$ where values are set to the number of 3D points associated with each voxel [S1].
5.  **Bi-level Fusion:** Local-level fusion integrates latent codes from $V_{cst}$ into the global volume $V_w$ by weighted averaging at the local level [S1]. Global-level fusion optimizes the global volume $V_w$ via neural rendering to ensure globally consistent reconstruction [S1].
6.  **SDF Query (Inference/Rendering):** To recover the SDF value of a 3D point $x$, retrieve 8 neighboring voxels $v_0...v_7$ in the volume grid [S1]. Transform $x$ to local coordinates relative to each neighbor: $\bar{x}_i = x - p_i$ [S1]. Compute the SDF using the weighted sum of decoder outputs:
    $$ s = \frac{\sum_{i=0}^{7} w(\bar{\bm{x}}_i, \bm{p}_i) \mathcal{D}(\bm{l}_i, \bar{\bm{x}}_i)}{\sum_{i=0}^{7} w(\bar{\bm{x}}_i, \bm{p}_i)} $$
    where $w(\cdot, \cdot)$ is the trilinear interpolation weight and $\mathcal{D}$ is the Shape Decoder [S1].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import math

class DepthEncoder:
    """Modified PointNet encoder for local shape embedding."""
    def __init__(self):
        # 4 fully connected layers: [128, 128, 128, 8]
        self.weights = [
            [[0.0]*3 for _ in range(128)],
            [[0.0]*128 for _ in range(128)],
            [[0.0]*128 for _ in range(128)],
            [[0.0]*128 for _ in range(8)]
        ]
        self.biases = [[0.0]*128, [0.0]*128, [0.0]*128, [0.0]*8]

    def forward(self, points):
        # points: list of [x, y, z]
        x = points
        for i in range(3):
            x = self._fc(x, self.weights[i], self.biases[i])
            x = [max(0.0, v) for v in x] # ReLU
        # Last layer: no activation
        x = self._fc(x, self.weights[3], self.biases[3])
        # Average pooling
        dim = len(x)
        return [sum(v) / dim for v in zip(*x)]

    def _fc(self, inp, W, b):
        out_dim = len(b)
        inp_dim = len(inp)
        return [sum(inp[j] * W[j][i] for j in range(inp_dim)) + b[i] for i in range(out_dim)]

class ShapeDecoder:
    """MLP decoder for SDF prediction."""
    def __init__(self):
        # 4 fully connected layers: [128, 128, 128, 1]
        # Input is concatenation of latent code (8) and local coord (3) -> 11
        self.weights = [
            [[0.0]*11 for _ in range(128)],
            [[0.0]*128 for _ in range(128)],
            [[0.0]*128 for _ in range(128)],
            [[0.0]*128 for _ in range(1)]
        ]
        self.biases = [[0.0]*128, [0.0]*128, [0.0]*128, [0.0]*1]

    def forward(self, latent_code, local_coord):
        x = latent_code + local_coord
        for i in range(3):
            x = self._fc(x, self.weights[i], self.biases[i])
            x = [max(0.0, v) for v in x] # ReLU
        return self._fc(x, self.weights[3], self.biases[3])[0] # [G]: Standard MLP activation not specified for decoder in sources

    def _fc(self, inp, W, b):
        out_dim = len(b)
        inp_dim = len(inp)
        return [sum(inp[j] * W[j][i] for j in range(inp_dim)) + b[i] for i in range(out_dim)]

def trilinear_weight(local_coord, voxel_size):
    # local_coord is x_bar_i = x - p_i
    # Returns weight for trilinear interpolation
    dist = [c / voxel_size for c in local_coord]
    dist = [max(-1.0, min(1.0, d)) for d in dist]
    return (1.0 - abs(dist[0])) * (1.0 - abs(dist[1])) * (1.0 - abs(dist[2]))

def get_8_neighbors(x, grid_origin, voxel_size, voxel_grid):
    # Returns list of 8 voxel objects
    cell = [int((x[i] - grid_origin[i]) / voxel_size) for i in range(3)]
    neighbors = []
    for dx in [-1, 0, 1]:
        for dy in [-1, 0, 1]:
            for dz in [-1, 0, 1]:
                idx = (cell[0]+dx, cell[1]+dy, cell[2]+dz)
                if idx in voxel_grid:
                    neighbors.append(voxel_grid[idx])
    return neighbors

def query_sdf(x, grid_origin, voxel_size, voxel_grid, encoder, decoder):
    neighbors = get_8_neighbors(x, grid_origin, voxel_size, voxel_grid)
    numerator = 0.0
    denominator = 0.0
    for v in neighbors:
        p_i = v['p']
        l_i = v['l']
        x_bar_i = [x[i] - p_i[i] for i in range(3)]
        w = trilinear_weight(x_bar_i, voxel_size)
        sdf_val = decoder.forward(l_i, x_bar_i)
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
*   **Time and Memory Complexity:** Not given in the sources.
*   **Measured Performance:** Evaluated on 3D Scene Dataset, ICL-NUIM RGBD benchmark, and ScanNet [S1].
*   **Strengths:** Compact representation due to latent codes; effective at capturing local geometric details; supports incremental updates via bi-level fusion [S1].
*   **Limitations:** Not given in the sources.
*   **When to use:** Not given in the sources.
*   **When not to use:** Not given in the sources.

## Background
The method was introduced in "BNV-Fusion: Dense 3D Reconstruction using Bi-level Neural Volume Fusion" [S1]. It builds on the concept of Neural Voxel Fields, extending them with a learned local shape embedding to improve geometric accuracy [S1]. The encoder is adapted from PointNet, a pioneering network for point cloud processing [S1]. The method was evaluated on ShapeNet CAD models, 3D Scene Dataset, ICL-NUIM, and ScanNet [S1].

## Sources
[S1] BNV-Fusion: Dense 3D Reconstruction using Bi-level Neural Volume Fusion, https://doi.org/10.1109/CVPR52688.2022.00607, fetched 2026-10-30
