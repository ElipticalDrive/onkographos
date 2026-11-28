# Neural Deformable Anchor (NeuDA)


## Overview
NeuDA is a neural implicit representation for high-fidelity surface reconstruction that replaces learnable feature vectors at voxel vertices with optimized 3D positions, called anchors, to adaptively encode local geometry [S1]. It solves the limitation of standard voxel grids where uniformly distributed features fail to capture small structures with complicated topologies or sharp local details [S1]. Unlike [Instant-NGP](18147-neural-duplex-radiance-fields-ndrf.md) which uses multi-resolution hash encoding and may suffer from hash collision, NeuDA uses hierarchical voxel grids with deformable anchors to better exploit spatial context [S1].

## How it works
**Inputs and Outputs:**
*   **Input:** A set of multi-view images, camera poses, and a hierarchical voxel grid structure $G$.
*   **Output:** A rendered color image $\hat{C}$ and a reconstructed surface mesh (extracted from the SDF).

**Data Structures:**
*   **Hierarchical Anchor Grid ($G$):** A set of levels. Each level consists of a grid where vertices store deformable anchors.
*   **Deformable Anchor ($\psi(G)$):** A set of parameters $\{p_v, \Delta p_v \mid v \in G\}$, where $p_v$ is the base position (initially uniform voxel vertex) and $\Delta p_v$ is the learnable offset [S1].
*   **SDF Network ($F$):** A 4-layer Multi-Layer Perceptron (MLP) taking geometric features and outputting signed distance $f(x)$ and geometric feature vector $z(x)$ [S1].
*   **Color Network ($M$):** A 4-layer MLP taking view direction $d$, normal $n$, and geometric feature $\hat{z}$ to output color $c$ [S1].

**Numbered Steps:**

1.  **Coordinate Normalization:** For a sample point $p$ along a ray, normalize its coordinate to the grid's scale, denoted as $x$ [S1].
2.  **Voxel Mapping:** Map the normalized coordinate $x$ to a specific voxel $V$ in the grid using the formula:
    $$ V = \{v \mid \lfloor x \cdot N \rfloor \le v < \lceil x \cdot N \rceil \} $$
    where $N$ is the grid size. The anchors are stored at the eight vertices of this voxel [S1].
3.  **Feature Interpolation:** Compute the input feature $\phi(p, \psi(G))$ by interpolating the frequency-encoded positions of the eight adjacent anchors:
    $$ \phi(p, \psi(G)) = \sum_{v \in V} w(p_v) \cdot \gamma(p_v + \Delta p_v) $$
    where $\psi(G) = \{p_v, \Delta p_v \mid v \in G\}$ is the set of deformable anchors, and $\gamma$ is the frequency encoding function [S1].
4.  **Frequency Encoding:** The frequency encoding function $\gamma$ is applied to the three coordinate values of the anchor position individually. For a hierarchical positional encoding (HPE) at level $l$, the function is:
    $$ \gamma(p_l) = (\sin(2^l \pi p_l), \cos(2^l \pi p_l)) $$
    This returns a small 6-dimensional embedding for each anchor grid level. Multi-level embeddings are concatenated [S1].
5.  **Weight Calculation:** The weight $w(p_n)$ for each anchor $p_n$ is calculated using cosine similarity to measure the contribution of different anchors:
    $$ w(p_n) = \frac{\hat{w}(p_n)}{\sum_n \hat{w}(p_n)}, \quad \hat{w}(p_n) = \frac{p \cdot p_n}{\|p\| \|p_n\|} $$
    where $p$ is the sample point and $p_n$ is the anchor position [S1].
6.  **SDF and Feature Prediction:** Feed the interpolated feature into the SDF network $F$ to obtain the signed distance $f(p)$ and geometric feature $z(p)$ [S1].
7.  **Opacity Calculation:** Compute the discrete opacity $\alpha_i$ for the sample point $t_i$ along the ray based on the SDF $f(p(t))$ and a trainable parameter $s$:
    $$ \alpha_i = \max\left( \frac{\Phi_s(f(p(t_i))) - \Phi_s(f(p(t_{i+1})))}{\Phi_s(f(p(t_i)))}, 0 \right) $$
    where $\Phi_s(x) = (1 + e^{-sx})^{-1}$ [S1].
8.  **Color Prediction:** Compute the surface normal from the SDF gradient. Feed the view direction $d$, normal $n$, and geometric feature $\hat{z} = z(x; \theta)$ into the color network $M$ to get color $c_i$ [S1].
9.  **Volume Rendering:** Accumulate the rendered color $\hat{C}$ along the ray with $N$ discrete sampled points:
    $$ \hat{C} = \sum_{i=1}^{N} T_i \alpha_i c_i $$
    where $T_i = \prod_{j=1}^{i-1}(1-\alpha_j)$ is the accumulated transmittance [S1].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import math
import torch

def neuda_forward(sample_points, ray_directions, camera_centers, grid_size, anchor_offsets, anchor_base_positions, sdf_mlp, color_mlp, num_samples):
    """
    NeuDA forward pass.
    
    Args:
        sample_points: Tensor of shape (batch_size, num_samples, 3), points along rays.
        ray_directions: Tensor of shape (batch_size, 3).
        camera_centers: Tensor of shape (batch_size, 3).
        grid_size: Integer N, the resolution of the voxel grid.
        anchor_offsets: Tensor of shape (num_anchors, 3), learnable offsets Delta p_v.
        anchor_base_positions: Tensor of shape (num_anchors, 3), base positions p_v.
        sdf_mlp: 4-layer MLP for SDF and geometric features.
        color_mlp: 4-layer MLP for color.
        num_samples: Integer N, number of samples per ray.
        
    Returns:
        rendered_colors: Tensor of shape (batch_size, 3).
    """
    batch_size = sample_points.shape[0]
    rendered_colors = torch.zeros(batch_size, 3).to(sample_points.device)
    
    for b in range(batch_size):
        ray_o = camera_centers[b]
        ray_d = ray_directions[b]
        points = sample_points[b] # Shape: (num_samples, 3)
        
        transmittance = 1.0
        accumulated_color = torch.zeros(3).to(points.device)
        
        for i in range(num_samples):
            p = points[i]
            
            # Step 1: Normalize coordinate to grid scale
            # Note: The paper defines x as normalized coordinate. 
            # Assuming grid spans [-1, 1] or similar, normalization depends on specific implementation.
            # Based on formula V = {v | floor(x*N) <= v < ceil(x*N)}, x is likely in [0, 1] or scaled to grid dimensions.
            # We assume a helper `normalize_to_grid` exists or is implicit in the mapping.
            x = normalize_to_grid(p, grid_size) 
            
            # Step 2: Map to voxel V
            # Find the 8 neighboring voxel vertices
            voxel_indices = get_voxel_vertices(x, grid_size)
            
            # Step 3 & 5: Interpolate features using deformable anchors
            # Get base positions and offsets for the 8 vertices
            # anchor_base_positions and anchor_offsets are indexed by vertex ID
            base_pos_8 = anchor_base_positions[voxel_indices] # Shape: (8, 3)
            offset_8 = anchor_offsets[voxel_indices]          # Shape: (8, 3)
            
            # Deformable positions: p_v + Delta p_v
            deformable_pos_8 = base_pos_8 + offset_8
            
            # Calculate weights using cosine similarity between sample point p and anchor positions
            # Note: The paper uses p (sample point) and p_n (anchor position).
            # Cosine similarity: (p . p_n) / (||p|| ||p_n||)
            # We must handle ||p|| = 0 or ||p_n|| = 0 to avoid division by zero [G]
            p_norm = torch.norm(p)
            if p_norm < 1e-8:
                weights = torch.ones(8) / 8.0
            else:
                # Dot products
                dot_products = torch.sum(p * deformable_pos_8, dim=1)
                # Norms of anchors
                anchor_norms = torch.norm(deformable_pos_8, dim=1)
                
                # Avoid division by zero for anchor norms [G]
                safe_anchor_norms = torch.where(anchor_norms < 1e-8, torch.ones_like(anchor_norms), anchor_norms)
                
                raw_weights = dot_products / (p_norm * safe_anchor_norms)
                
                # Normalize weights
                weights = raw_weights / (torch.sum(raw_weights) + 1e-8)
            
            # Step 4: Frequency Encoding
            # Apply hierarchical positional encoding to each deformable anchor position
            # The paper mentions HPE policy. For simplicity, we use the basic frequency encoding defined.
            # gamma(p_l) = (sin(2^l * pi * p_l), cos(2^l * pi * p_l))
            # Assuming single level for this basic implementation, or concatenating levels.
            # We assume a function `frequency_encode` that handles the sin/cos expansion.
            encoded_anchors = frequency_encode(deformable_pos_8) # Shape: (8, encoded_dim)
            
            # Interpolate: phi = sum(w * gamma)
            # weights shape: (8,), encoded_anchors shape: (8, encoded_dim)
            feature = torch.sum(weights.unsqueeze(1) * encoded_anchors, dim=0) # Shape: (encoded_dim,)
            
            # Step 6: SDF and Feature Prediction
            sdf_val, geo_feature = sdf_mlp(feature) # Returns scalar SDF and vector z
            
            # Step 7: Opacity Calculation
            # Phi_s(x) = (1 + exp(-s*x))^-1
            # We need SDF at current point t_i and next point t_{i+1}
            # For the last sample, opacity is typically 1 or handled via boundary condition [G]
            if i < num_samples - 1:
                p_next = points[i+1]
                x_next = normalize_to_grid(p_next, grid_size)
                voxel_indices_next = get_voxel_vertices(x_next, grid_size)
                base_pos_8_next = anchor_base_positions[voxel_indices_next]
                offset_8_next = anchor_offsets[voxel_indices_next]
                deformable_pos_8_next = base_pos_8_next + offset_8_next
                
                # Re-calculate weights for next point
                p_next_norm = torch.norm(p_next)
                if p_next_norm < 1e-8:
                    weights_next = torch.ones(8) / 8.0
                else:
                    dot_products_next = torch.sum(p_next * deformable_pos_8_next, dim=1)
                    anchor_norms_next = torch.norm(deformable_pos_8_next, dim=1)
                    safe_anchor_norms_next = torch.where(anchor_norms_next < 1e-8, torch.ones_like(anchor_norms_next), anchor_norms_next)
                    raw_weights_next = dot_products_next / (p_next_norm * safe_anchor_norms_next)
                    weights_next = raw_weights_next / (torch.sum(raw_weights_next) + 1e-8)
                
                encoded_anchors_next = frequency_encode(deformable_pos_8_next)
                feature_next = torch.sum(weights_next.unsqueeze(1) * encoded_anchors_next, dim=0)
                sdf_val_next, _ = sdf_mlp(feature_next)
                
                # Phi_s function
                phi_s_curr = 1.0 / (1.0 + torch.exp(-s * sdf_val))
                phi_s_next = 1.0 / (1.0 + torch.exp(-s * sdf_val_next))
                
                # Alpha calculation
                numerator = phi_s_curr - phi_s_next
                denominator = phi_s_curr
                alpha = torch.clamp(numerator / (denominator + 1e-8), min=0.0)
            else:
                # Last sample, assume full opacity if inside surface or handle differently [G]
                # Standard volume rendering often sets alpha=1 for the last sample to ensure convergence
                alpha = 1.0
            
            # Step 8: Color Prediction
            # Compute normal from SDF gradient (not shown in formula, but required for color network)
            # In practice, this requires backpropagation or finite differences. 
            # For forward pass pseudocode, we assume a `compute_normal` function exists.
            normal = compute_normal(sdf_mlp, feature, p)
            
            # Color network inputs: view direction d, normal n, geometric feature z
            # z is geo_feature from SDF network
            color = color_mlp(ray_directions[b], normal, geo_feature)
            
            # Step 9: Accumulate
            accumulated_color += transmittance * alpha * color
            transmittance *= (1.0 - alpha)
            
            if transmittance < 1e-4:
                break
                
        rendered_colors[b] = accumulated_color
        
    return rendered_colors

def normalize_to_grid(point, grid_size):
    """
    Normalizes a 3D point to the grid scale [0, 1] assuming the grid spans [-1, 1].
    This is a placeholder for the specific normalization used in the paper.
    """
    # Assuming point is in [-1, 1]
    return (point + 1.0) / 2.0

def get_voxel_vertices(normalized_point, grid_size):
    """
    Returns the indices of the 8 vertices of the voxel containing the normalized point.
    """
    # normalized_point is in [0, 1]
    # Grid indices are 0 to grid_size-1
    coords = normalized_point * grid_size
    floor_coords = torch.floor(coords).long()
    ceil_coords = torch.ceil(coords).long()
    
    # Clamp to grid boundaries
    floor_coords = torch.clamp(floor_coords, 0, grid_size - 1)
    ceil_coords = torch.clamp(ceil_coords, 0, grid_size - 1)
    
    # Generate 8 combinations
    vertices = []
    for x in [floor_coords[0], ceil_coords[0]]:
        for y in [floor_coords[1], ceil_coords[1]]:
            for z in [floor_coords[2], ceil_coords[2]]:
                vertices.append(x * grid_size * grid_size + y * grid_size + z)
    return torch.tensor(vertices, dtype=torch.long)

def frequency_encode(positions):
    """
    Applies frequency encoding to positions.
    gamma(p_l) = (sin(2^l * pi * p_l), cos(2^l * pi * p_l))
    Assuming single level l=0 for basic implementation.
    """
    # positions shape: (num_anchors, 3)
    # Output shape: (num_anchors, 6)
    sin_vals = torch.sin(2 * math.pi * positions)
    cos_vals = torch.cos(2 * math.pi * positions)
    return torch.cat([sin_vals, cos_vals], dim=1)

def compute_normal(sdf_mlp, feature, point):
    """
    Computes the surface normal using finite differences.
    """
    eps = 1e-4
    dx = torch.zeros_like(point)
    dx[0] = eps
    dy = torch.zeros_like(point)
    dy[1] = eps
    dz = torch.zeros_like(point)
    dz[2] = eps
    
    # Perturb point and re-encode to get SDF
    # This is computationally expensive and usually done via autograd in PyTorch
    # Here we simulate the gradient
    # In a real implementation, use torch.autograd.grad
    return torch.zeros(3) # Placeholder
```

**Parameters:**
*   **Grid Size ($N$):** Not given in the sources.
*   **MLP Layers:** The SDF network $F$ is a 4-layer MLP. The color network $M$ is a 4-layer MLP [S1].
*   **Trainable Parameter ($s$):** Used in the sigmoid approximation $\Phi_s$. $1/s$ approaches zero as optimization converges [S1].
*   **Frequency Encoding:** Uses hierarchical positional encoding (HPE) where higher frequency bands are applied to finer grid levels [S1].

**Edge Cases and Pitfalls:**
*   **Uniform Distribution Limitation:** Standard voxel grids store uniformly distributed features which may fail to capture small structures with complicated topologies. NeuDA solves this by optimizing anchor positions to move from corners to the vicinity of abrupt geometry-changing areas [S1].
*   **Hash Collision:** Unlike Instant-NGP which uses multi-resolution hash encoding and may suffer from hash collision, NeuDA uses hierarchical voxel grids with deformable anchors [S1].
*   **Precision:** The weight calculation involves division by norms. Care must be taken to avoid division by zero if the sample point or anchor position is at the origin [G].

## Operating characteristics
*   **Time Complexity:** Not given in the sources.
*   **Memory Complexity:** Not given in the sources.
*   **Measured Performance:** NeuDA employs a shallower MLP (4 layers vs. 8 layers for NeuS and volSDF) to achieve better surface reconstruction performance due to the promising scene representation capability of the hierarchical deformable anchor structure [S1].
*   **Strengths:** Improves flexibility of voxel grid representation, better exploits spatial context in 3D space, and captures sharp local topologies such as small holes and structures [S1].
*   **Limitations:** Not given in the sources.
*   **When to use:** When high-fidelity surface reconstruction is required, especially for scenes with small structures and complicated topologies, and when avoiding hash collisions associated with hash-based encodings is desired [S1].
*   **When not to use:** Not given in the sources.

## Background
NeuDA was introduced by Bowen Cai, Jinchi Huang, Rongfei Jia, Chengfei Lv, and Huan Fu from Alibaba Group. It was published on arXiv on March 26, 2023 (arXiv:2303.02375v2) [S1]. The method builds on [NeuS](18147-neural-duplex-radiance-fields-ndrf.md) but replaces the fixed feature vectors with deformable anchors to address the limitation of overlooking spatial context in 3D space [S1].

## Sources
[S1] NeuDA: Neural Deformable Anchor for High-Fidelity Implicit Surface Reconstruction, https://doi.org/10.1109/cvpr52729.2023.00819, fetched 2026-10-26
