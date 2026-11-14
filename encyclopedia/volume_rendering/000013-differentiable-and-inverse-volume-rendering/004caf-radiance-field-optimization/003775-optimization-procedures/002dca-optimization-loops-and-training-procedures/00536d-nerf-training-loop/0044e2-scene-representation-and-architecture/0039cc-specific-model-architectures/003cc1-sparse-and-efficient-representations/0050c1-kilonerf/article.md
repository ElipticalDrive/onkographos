# KiloNeRF

Also called: KiloNeRF: Speeding up Neural Radiance Fields with Thousands of Tiny MLPs

## Overview
KiloNeRF accelerates Neural Radiance Field (NeRF) rendering by three orders of magnitude without sacrificing visual quality by replacing the single large MLP with thousands of independent, small MLPs arranged on a regular 3D grid [S2]. Unlike the original NeRF, which queries one deep network for every sample point, KiloNeRF maps 3D positions to specific grid cells, each governed by a lightweight network [S2]. This spatial decomposition allows for significant speedups compared to both the original NeRF and other sparse methods like Neural Sparse Voxel Fields (NSVF) [S2].

## How it works
**Inputs and Outputs:**
*   **Inputs:** A set of $N$ calibrated images with known camera poses; an axis-aligned bounding box (AABB) defined by minimum bounds $\mathbf{b}_{min}$ and maximum bounds $\mathbf{b}_{max}$ enclosing the scene.
*   **Outputs:** A set of $N_{cells}$ independent MLPs (the "KiloNeRF" model) and an occupancy grid for acceleration. During inference, given a ray $\mathbf{r}$, the output is the synthesized pixel color $C(\mathbf{r})$.

**Data Structures:**
1.  **Network Grid:** A uniform 3D grid of resolution $\mathbf{r} = (r_x, r_y, r_z)$ within the AABB. Each cell index $\mathbf{i} = (i_x, i_y, i_z)$ corresponds to a unique MLP with parameters $\theta_{\mathbf{i}}$. The maximum resolution is $16 \times 16 \times 16 = 4096$ networks [S2].
2.  **Occupancy Grid:** A second uniform grid with resolution $16 \times \mathbf{r}$ (upper bounded by $256 \times 256 \times 256$). Each cell contains a binary value indicating if the scene has content [S2].
3.  **Tiny MLP Architecture:**
    *   **Input:** Concatenation of positional encoding $\gamma(\mathbf{x})$ and viewing direction $\mathbf{d}$.
    *   **Layers:** 4 hidden layers, each with 32 hidden units.
    *   **Activations:** All affine layers use ReLU, except:
        *   The input to the penultimate hidden layer has no activation [S2].
        *   The output layer for RGB color $\mathbf{c}$ uses Sigmoid activation [S2].
    *   **Structure:** No skip connections. The viewing direction $\mathbf{d}$ is provided as an additional input to the last hidden layer [S2].
    *   **Outputs:** Density $\sigma$ (scalar, independent of $\mathbf{d}$) and Color $\mathbf{c}$ (RGB vector) [S2].

**Numbered Steps:**

**Stage 1: Teacher Training**
1.  Train a standard NeRF (the "teacher") on the training images for up to 600,000 iterations [S2]. This teacher uses hierarchical sampling and a deep MLP (10 hidden layers, 256/128 units) [S2].

**Stage 2: Occupancy Grid Extraction**
2.  Instantiate the occupancy grid with resolution $16 \times \mathbf{r}$ [S2].
3.  For each cell in the occupancy grid, sample densities on a $3 \times 3 \times 3$ subgrid using the teacher NeRF [S2].
4.  Mark a cell as occupied if any evaluated density $\sigma > \tau$, where $\tau = 10$ [S2].

**Stage 3: Distillation Training**
5.  Initialize the KiloNeRF networks $\theta_{\mathbf{i}}$ randomly or with small values [G].
6.  For each network $\mathbf{i}$, randomly sample a batch of 3D points $\mathbf{x}$ inside its corresponding grid cell [S2].
7.  Augment these points with viewing directions $\mathbf{d}$ drawn randomly from the unit sphere [S2].
8.  Query the teacher NeRF to obtain target density $\sigma_{teacher}$ and color $\mathbf{c}_{teacher}$ [S2].
9.  Query the student KiloNeRF network $\theta_{\mathbf{i}}$ to obtain $\sigma_{student}$ and $\mathbf{c}_{student}$ [S2].
10. Convert densities to alpha values using $\alpha = 1 - \exp(-\sigma \delta)$, where $\delta$ is a fixed distance between samples (e.g., derived from the grid cell size or a constant) [S2].
11. Optimize $\theta_{\mathbf{i}}$ using an L2 loss between the student's $(\alpha_{student}, \mathbf{c}_{student})$ and the teacher's $(\alpha_{teacher}, \mathbf{c}_{teacher})$ [S2].
12. Apply L2 regularization with weight $\lambda = 10^{-6}$ to the weights and biases of the last two layers of the student networks [S2].
13. Repeat for 150,000 iterations [S2].

**Stage 4: Fine-tuning**
14. Fine-tune all KiloNeRF networks on the original training images using photometric loss (L2 loss between rendered color and ground truth) for 1,000,000 iterations [S2].
15. Use the occupancy grid for Empty Space Skipping (ESS) during this stage [S2].

**Inference (Rendering):**
16. For each ray $\mathbf{r}$, sample up to $K=384$ equidistant points [S2].
17. **Empty Space Skipping (ESS):** For each sample point $\mathbf{x}$, determine its occupancy grid cell. Only query the KiloNeRF network if the cell is occupied [S2].
18. **Network Selection:** Determine the network index $\mathbf{i}$ for point $\mathbf{x}$ using spatial binning:
    $$
    \mathbf{i} = \left\lfloor \frac{\mathbf{x} - \mathbf{b}_{min}}{(\mathbf{b}_{max} - \mathbf{b}_{min}) / \mathbf{r}} \right\rfloor
    $$
    where operations are element-wise [S2].
19. Evaluate the selected network: $(\mathbf{c}, \sigma) = f_{\theta_{\mathbf{i}}}(\gamma(\mathbf{x}), \mathbf{d})$ [S2].
20. **Early Ray Termination (ERT):** During inference, group queries by distance from the camera. If the transmittance $T_i < \epsilon$ (where $\epsilon = 0.01$), stop sampling further points for that ray [S2].
21. Compute pixel color via volumetric rendering:
    $$
    C(\mathbf{r}) = \sum_{i=1}^{N} T_i \alpha_i \mathbf{c}_i, \quad \alpha_i = 1 - \exp(-\sigma_i \delta_i), \quad T_i = \prod_{j=1}^{i-1} (1 - \alpha_j)
    $$
    where $\delta_i$ is the distance between samples [S2].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import torch
import torch.nn as nn
import math

class TinyMLP(nn.Module):
    """
    KiloNeRF Tiny MLP: 4 hidden layers, 32 units.
    Input: Encoded Position (3D) + Viewing Direction (3D)
    Output: Density (1), Color (3)
    """
    def __init__(self, pos_enc_dim, view_enc_dim):
        super(TinyMLP, self).__init__()
        # Input: pos_enc_dim + view_enc_dim
        # Layer 1
        self.fc1 = nn.Linear(pos_enc_dim + view_enc_dim, 32)
        # Layer 2
        self.fc2 = nn.Linear(32, 32)
        # Layer 3
        self.fc3 = nn.Linear(32, 32)
        # Layer 4 (Penultimate)
        self.fc4 = nn.Linear(32, 32)
        # No activation on input to penultimate layer? 
        # Note: Source says "no activation is applied to the feature vector 
        # that is given as input to the penultimate hidden layer". 
        # This implies the output of fc3 is passed directly to fc4 without ReLU.
        
        # Density Head (from Layer 2 output? Or Layer 4?)
        # Source: "density is independent of view direction" and "viewing direction 
        # is provided as an additional input to the last hidden layer".
        # NeRF structure: Density head comes from earlier layers.
        # KiloNeRF Source: "Like in NeRF, we provide the viewing direction as an 
        # additional input to the last hidden layer."
        # "All affine layers are followed by a ReLU activation with two exceptions: 
        # The output layer that computes the RGB color c uses a sigmoid activation 
        # and no activation is applied to the feature vector that is given as input 
        # to the penultimate hidden layer."
        # This implies:
        # x -> fc1 -> ReLU -> fc2 -> ReLU -> fc3 -> (no relu) -> fc4 -> ReLU -> fc5?
        # Wait, "4 hidden layers".
        # Let's trace NeRF: Input -> H1(256) -> H2(256) -> ... -> H9(256) -> H10(128) -> Density/Color.
        # KiloNeRF: 4 hidden layers.
        # Input -> H1(32) -> H2(32) -> H3(32) -> H4(32) -> Output.
        # Where does density come from?
        # Source: "density is independent of view direction".
        # In NeRF, density head is attached to H9 (or similar).
        # In KiloNeRF, since view direction is only added to the LAST hidden layer,
        # the density head must be attached to an earlier layer (e.g., H2 or H3) 
        # that does not see the view direction.
        # Source Fig 2 description: "enforces that the predicted density is independent 
        # of the view direction... viewing direction as an additional input to the last hidden layer."
        # So:
        # Path 1 (Density): Input -> H1 -> ReLU -> H2 -> ReLU -> Density Head.
        # Path 2 (Color): Input -> H1 -> ReLU -> H2 -> ReLU -> H3 -> (no relu) -> H4 -> ReLU -> Color Head.
        # View Direction is concatenated before H4.
        
        self.density_head = nn.Linear(32, 1)
        self.color_head = nn.Linear(32 + view_enc_dim, 3) # Input to last layer includes view dir

    def forward(self, pos_enc, view_dir):
        # H1
        x = self.fc1(pos_enc)
        x = torch.relu(x)
        
        # H2
        x = self.fc2(x)
        x = torch.relu(x)
        
        # Density Output (from H2, before view dir is added)
        sigma = self.density_head(x)
        
        # H3
        x = self.fc3(x)
        # No activation on input to penultimate (H4)
        
        # H4 (Penultimate)
        # Concatenate view direction
        x_with_view = torch.cat([x, view_dir], dim=-1)
        x = self.fc4(x_with_view)
        x = torch.relu(x)
        
        # Color Output
        c = torch.sigmoid(self.color_head(x))
        
        return sigma, c

def get_network_index(x, b_min, b_max, r):
    """
    Map 3D position x to grid cell index i.
    """
    # Element-wise floor division
    cell_size = (b_max - b_min) / r
    i = torch.floor((x - b_min) / cell_size)
    return i.long()

def render_ray(ray_origin, ray_direction, b_min, b_max, r, networks, occupancy_grid, K=384, epsilon=0.01):
    """
    Render a single ray using KiloNeRF.
    """
    # 1. Sample points
    # Equidistant sampling along the ray within the AABB
    # Calculate intersection with AABB to get t_min, t_max
    # ... (standard ray-AABB intersection) ...
    t_vals = torch.linspace(t_min, t_max, K)
    points = ray_origin.unsqueeze(1) + ray_direction.unsqueeze(1) * t_vals.unsqueeze(0)
    
    # 2. Empty Space Skipping & 3. Early Ray Termination
    # Sort points by distance from camera (already sorted by t_vals)
    transmittance = 1.0
    color = torch.zeros(3)
    
    # Pre-compute occupancy for all points to avoid repeated lookups if possible
    # Or check on the fly
    
    for i, point in enumerate(points):
        # Check Transmittance for ERT
        if transmittance < epsilon:
            break
            
        # Get Grid Index
        idx = get_network_index(point, b_min, b_max, r)
        
        # Check Occupancy Grid
        if not occupancy_grid[idx]:
            continue
            
        # Get Network
        net = networks[idx]
        
        # Positional Encoding (Simplified placeholder for gamma)
        pos_enc = positional_encoding(point, L=10) # L matches NeRF
        view_enc = positional_encoding(ray_direction, L=4) # Standard for view dir
        
        # Query Network
        sigma, c = net(pos_enc, view_enc)
        
        # Convert sigma to alpha
        # Delta is distance between samples. For equidistant, delta = (t_max - t_min) / K
        # However, source says "K controls distance delta". 
        # In implementation, delta_i is usually t_{i+1} - t_i.
        delta = t_vals[i+1] - t_vals[i] if i < K-1 else t_vals[i] - t_vals[i-1]
        # Handle boundary cases for delta
        
        alpha = 1.0 - torch.exp(-sigma * delta)
        
        # Accumulate
        weight = transmittance * alpha
        color += weight * c
        transmittance *= (1.0 - alpha)
        
    return color

```

**Parameters:**
*   **Grid Resolution:** Largest dimension 16; others chosen for cubic cells. Max 4096 networks [S2].
*   **Occupancy Grid Resolution:** $16 \times$ network grid resolution. Max $256^3$ [S2].
*   **Occupancy Threshold $\tau$:** 10 [S2].
*   **Max Samples $K$:** 384 [S2].
*   **ERT Threshold $\epsilon$:** 0.01 [S2].
*   **L2 Regularization Weight:** $10^{-6}$ on last two layers [S2].
*   **Optimizer:** Adam, LR $5 \times 10^{-4}$, batch size 8192 pixels [S2].

**Edge Cases and Pitfalls:**
*   **Free Space Artifacts:** Training KiloNeRF from scratch leads to noticeable artifacts in empty space. This is mitigated by distillation from a pre-trained teacher [S2].
*   **View-Dependent Bias:** Reducing output features of the last hidden layer to mimic NeRF's view-dependent bias caused visual quality loss due to small network width. This is solved by L2 regularization on the last two layers [S2].
*   **Dynamic Parameter Loading:** During rendering, a ray traverses different networks, requiring dynamic parameter loading. This necessitates custom CUDA kernels or efficient batched matrix multiplication libraries (like MAGMA) to fuse evaluations [S2].
*   **Memory Overhead:** Deploying thousands of MLPs incurs substantial memory overhead compared to a single network [S3].

## Operating characteristics
*   **Speedup:** KiloNeRF achieves a speedup of 2000x to 2548x compared to the original NeRF [S1, S2]. It is approximately two orders of magnitude faster than Neural Sparse Voxel Fields (NSVF) [S2].
*   **Rendering Time:** On an NVIDIA GTX 1080 Ti, KiloNeRF renders in 30ms (BlendedMVS), 26ms (Synthetic-NeRF), 26ms (Synthetic-NSVF), and 91ms (Tanks&Temples) [S2].
*   **Quality:** Visual quality (PSNR, SSIM, LPIPS) is comparable to the original NeRF and NSVF [S2].
*   **Memory:** Higher memory usage than NeRF due to the storage of thousands of small MLPs [S3].
*   **Strengths:** Real-time capable rendering; no loss in visual quality; simpler spatial decomposition than DeRF (regular grid vs. Voronoi) [S2].
*   **Limitations:** Requires an AABB; distillation adds training complexity; dynamic network switching requires specialized implementation for optimal speed [S2].

## Background
KiloNeRF was introduced by Reiser et al. in 2021 at ICCV [S2]. It addresses the slow rendering times of NeRF by decomposing the scene into thousands of small MLPs. The method builds on the teacher-student distillation paradigm and incorporates acceleration techniques like Empty Space Skipping and Early Ray Termination, which were previously used in NSVF [S2]. It is cited as a key method for accelerating NeRF by three orders of magnitude [S1, S3].

## Sources
[S1] Neural Radiance Fields in 3D Vision: A Comprehensive Review, https://doi.org/10.26599/cvm.2026.9450535, fetched 2026-10-28
[S2] KiloNeRF: Speeding up Neural Radiance Fields with Thousands of Tiny MLPs, https://doi.org/10.1109/iccv48922.2021.01407, fetched 2026-10-26
[S3] Recent Advances and Trends in Learning‐Based 3D Representations, https://doi.org/10.1111/cgf.70530, fetched 2026-10-26
[S4] Faster Neural Radiance Fields Inference — Chris Choy, https://chrischoy.org/posts/nerfs/, fetched 2026-10-29 (unofficial)
[S5] NeRF的基础及后续扩展, https://pengsida.net/files/NeRF_Tutorial.pdf, fetched 2026-10-30 (unofficial)
