# Neural Deformable Anchor (NeuDA)

## Overview
NeuDA is a neural implicit surface reconstruction method that replaces learnable feature vectors at voxel vertices with optimized 3D positions (anchors) to adaptively encode local geometry [S1]. It solves the limitation of standard voxel grids, where uniformly distributed features fail to capture small structures with complicated topologies, by allowing anchors to move from grid corners to the vicinity of abrupt geometry-changing areas [S1]. Unlike Instant-NGP, which uses multi-resolution hash encoding and may suffer from hash collisions, NeuDA uses hierarchical voxel grids with deformable anchors to better exploit spatial context [S1].

## How it works
**Inputs and Outputs:**
*   **Input:** Camera ray $r(t) = o + td$, where $o$ is the origin and $d$ is the direction. Hierarchical voxel grid $G$.
*   **Output:** Rendered color $\hat{C}$ and surface normal $\mathbf{n}$.

**Data Structures:**
*   **Deformable Anchors $\psi(G)$:** A set of parameters for each vertex $v$ in the grid $G$. Each vertex stores:
    *   $p_v \in \mathbb{R}^3$: The base 3D position of the anchor.
    *   $\Delta p_v \in \mathbb{R}^3$: The learnable offset for the anchor.
    *   Initially, anchors are uniformly distributed at voxel vertices [S1].
*   **Grid Levels:** Hierarchical levels $l$, where higher levels have finer resolution and higher frequency bands [S1].

**Numbered Steps:**

1.  **Ray Sampling:** Sample $N$ points $p(t_i)$ along the ray $r(t)$ within the near/far range [S1].
2.  **Anchor Interpolation (Feature Extraction):** For each sample point $p$:
    a. Normalize $p$ to the grid scale to get $x$.
    b. Identify the voxel $V$ containing $x$: $V = \{v \mid \lfloor x \cdot N \rfloor \le v < \lceil x \cdot N \rceil\}$, where $N$ is the grid size. The 8 vertices of $V$ are the adjacent anchors [S1].
    c. Compute the input feature $\phi(p, \psi(G))$ by interpolating the frequency-encoded positions of the 8 anchors:
       $$ \phi(p, \psi(G)) = \sum_{v \in V} w(p_v) \cdot \gamma(p_v + \Delta p_v) $$
       where $\psi(G) = \{p_v, \Delta p_v \mid v \in G\}$ is the set of deformable anchors [S1].
    d. The weight $w(p_v)$ is the cosine similarity between the sample point $p$ and the anchor position $p_v$:
       $$ w(p_v) = \frac{\hat{w}(p_v)}{\sum_{n} \hat{w}(p_n)}, \quad \hat{w}(p_v) = \frac{p \cdot p_v}{\|p\| \|p_v\|} $$
       [S1].
    e. The frequency encoding function $\gamma$ is applied to the three coordinate values of the deformed anchor position $(p_v + \Delta p_v)$ individually. For a scalar $p_l$ and level $l$:
       $$ \gamma(p_l) = (\sin(2^l \pi p_l), \cos(2^l \pi p_l)) $$
       [S1].
    f. Concatenate features from all hierarchical levels to form the final input feature vector [S1].
3.  **SDF Prediction:** Feed the feature vector into the SDF network $F$ (a 4-layer MLP) to predict the signed distance value $f(p)$ and the geometric feature $z(x)$ [S1].
4.  **Opacity Calculation:** Compute discrete opacity $\alpha_i$ for each sample point $t_i$ using the SDF values $f(p(t_i))$ and $f(p(t_{i+1}))$ and a trainable parameter $s$:
    $$ \alpha_i = \max\left( \frac{\Phi_s(f(p(t_i))) - \Phi_s(f(p(t_{i+1})))}{\Phi_s(f(p(t_i)))}, 0 \right) $$
    where $\Phi_s(x) = (1 + e^{-sx})^{-1}$ [S1].
5.  **Color Prediction:** Feed the geometric feature $z(x)$, view direction $d$, and estimated normal $\mathbf{n}$ into the color network $M$ (a 4-layer MLP) to predict RGB color $c_i$ [S1].
6.  **Volume Rendering:** Accumulate color along the ray:
    $$ \hat{C} = \sum_{i=1}^{N} T_i \alpha_i c_i, \quad T_i = \prod_{j=1}^{i-1}(1-\alpha_j) $$
    [S1].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import math

def frequency_encoding(pos, level):
    """
    Applies hierarchical positional encoding to a 3D position vector.
    pos: list of 3 floats [x, y, z]
    level: int, the grid level index
    Returns: list of floats representing the encoded vector
    """
    encoded = []
    for coord in pos:
        freq = 2**level * math.pi
        encoded.append(math.sin(freq * coord))
        encoded.append(math.cos(freq * coord))
    return encoded

def cosine_similarity_weight(p, p_anchor):
    """
    Computes normalized cosine similarity weight between sample point p and anchor p_anchor.
    p: list of 3 floats
    p_anchor: list of 3 floats
    Returns: float weight
    """
    dot_product = sum(a * b for a, b in zip(p, p_anchor))
    norm_p = math.sqrt(sum(a**2 for a in p))
    norm_anchor = math.sqrt(sum(a**2 for a in p_anchor))
    
    if norm_p == 0 or norm_anchor == 0:
        return 0.0
        
    raw_weight = dot_product / (norm_p * norm_anchor)
    return raw_weight

def render_ray(ray_origin, ray_direction, grid, levels, sdf_net, color_net, s_param):
    """
    Renders a single pixel by casting a ray through the NeuDA representation.
    """
    # 1. Sample points along the ray
    # Assume N samples are pre-defined or generated based on near/far bounds
    samples = []
    for i in range(N):
        t = near + (far - near) * (i / (N - 1))
        p = [ray_origin[j] + ray_direction[j] * t for j in range(3)]
        samples.append(p)
        
    accumulated_color = [0.0, 0.0, 0.0]
    accumulated_transmittance = 1.0
    
    for i, p in enumerate(samples):
        # 2. Anchor Interpolation
        # Normalize p to grid scale
        x = [p[j] / grid_scale for j in range(3)]
        
        # Find voxel vertices (8 neighbors)
        # floor and ceil indices
        idx_floor = [math.floor(x[j] * grid_size) for j in range(3)]
        idx_ceil = [math.ceil(x[j] * grid_size) for j in range(3)]
        
        # Generate all 8 vertex indices
        vertex_indices = []
        for dx in [0, 1]:
            for dy in [0, 1]:
                for dz in [0, 1]:
                    vx = idx_floor[0] + dx
                    vy = idx_floor[1] + dy
                    vz = idx_floor[2] + dz
                    vertex_indices.append((vx, vy, vz))
        
        # Interpolate features from anchors
        total_weight = 0.0
        feature_vector = []
        
        # Initialize feature vector for concatenation across levels
        # Note: In practice, features from different levels are concatenated.
        # Here we show the interpolation for a single level 'l'.
        for l in levels:
            level_feature = []
            for v_idx in vertex_indices:
                # Get anchor position and offset from grid
                p_v = grid.get_anchor_position(v_idx)
                delta_p_v = grid.get_anchor_offset(v_idx)
                
                # Deformed anchor position
                deformed_pos = [p_v[j] + delta_p_v[j] for j in range(3)]
                
                # Frequency encoding
                enc = frequency_encoding(deformed_pos, l)
                
                # Weight calculation
                w = cosine_similarity_weight(p, p_v)
                total_weight += w
                
                # Accumulate weighted feature
                if not level_feature:
                    level_feature = [w * e for e in enc]
                else:
                    level_feature = [level_feature[k] + w * enc[k] for k in range(len(enc))]
            
            # Normalize weights for this level
            if total_weight > 0:
                level_feature = [f / total_weight for f in level_feature]
            
            # Concatenate to main feature vector
            feature_vector.extend(level_feature)
            
        # 3. SDF Prediction
        # Feature vector is input to 4-layer MLP F
        sdf_val, geo_feat = sdf_net(feature_vector)
        
        # 4. Opacity Calculation
        # Need SDF at next point for alpha calculation
        if i < N - 1:
            p_next = samples[i+1]
            # Re-compute feature for p_next or cache it
            # For simplicity, assume we have sdf_val_next
            # In implementation, one would typically compute SDF for all points first
            pass 
            
        # Approximate alpha using current SDF and a small epsilon or next point
        # Using the formula from S1: alpha_i = max((Phi_s(f(t_i)) - Phi_s(f(t_{i+1}))) / Phi_s(f(t_i)), 0)
        # We need f(t_{i+1}). Let's assume we compute SDF for all points in a pre-pass or loop.
        # Here we demonstrate the logic for one point assuming f_next is available.
        
        phi_current = 1.0 / (1.0 + math.exp(-s_param * sdf_val))
        
        # Placeholder for next point SDF calculation in a real loop
        # In a single pass, one might compute alpha based on local gradient or 
        # use the previous point's transmittance logic differently. 
        # However, the paper defines alpha_i based on t_i and t_{i+1}.
        # We will assume a pre-calculation of SDFs for all t_i.
        
        # Let's assume we have a list of SDF values for all samples
        # alpha_i calculation requires f(t_i) and f(t_{i+1})
        # This step is usually done after SDF prediction for all points.
        
        # 5. Color Prediction
        # Estimate normal from SDF gradient or predict from network?
        # Paper says M takes normal n. Usually normal is derived from SDF gradient 
        # or predicted. NeuDA paper mentions M takes n. 
        # Let's assume normal is computed via finite differences or predicted.
        # For this pseudocode, we assume normal n is available.
        color = color_net(geo_feat, ray_direction, normal)
        
        # 6. Accumulation
        # T_i is product of (1-alpha_j) for j < i
        accumulated_color = [accumulated_color[k] + accumulated_transmittance * alpha * color[k] for k in range(3)]
        accumulated_transmittance *= (1 - alpha)
        
    return accumulated_color
```

**Parameters:**
*   $s$: Trainable parameter in the sigmoid function $\Phi_s$. As optimization converges, $1/s$ approaches zero [S1].
*   MLP Depth: SDF network $F$ is 4 layers; Color network $M$ is 4 layers [S1].

**Edge Cases and Pitfalls:**
*   **Hash Collisions:** Unlike hash-based methods (e.g., Instant-NGP), NeuDA avoids hash collisions by using hierarchical voxel grids [S1].
*   **Uniform Distribution Failure:** Standard voxel grids fail to capture small structures with complicated topologies because features are uniformly distributed. NeuDA mitigates this by optimizing anchor positions to adaptively encode local geometry [S1].
*   **Precision:** The weight calculation uses cosine similarity. If the sample point $p$ or anchor $p_v$ is at the origin (norm 0), the weight is undefined (handled as 0.0 in pseudocode) [S1].

## Operating characteristics
*   **Complexity:** Time and memory complexity are not explicitly given in the sources.
*   **Performance:** NeuDA employs a shallower MLP (4 layers vs. 8 layers for NeuS and volSDF) to achieve better surface reconstruction performance due to the promising scene representation capability of the hierarchical deformable anchor structure [S1].
*   **Strengths:** Superior in recovering high-quality geometry with fine-grained details compared to baselines and SOTA methods on DTU and BlendedMVS datasets [S1]. It better exploits spatial context in 3D space [S1].
*   **Limitations:** Not given in the sources.
*   **When to use:** When high-fidelity surface reconstruction with fine-grained details is required, especially for small structures with complicated topologies [S1].

## Background
NeuDA was introduced by Bowen Cai, Jinchi Huang, Rongfei Jia, Chengfei Lv, and Huan Fu from Alibaba Group. It was published on arXiv on March 26, 2023 (arXiv:2303.02375v2) [S1]. The method builds on the principles of differentiable volume rendering and neural implicit surface reconstruction, specifically addressing limitations in IDR and NeuS regarding spatial context and sharp local topologies [S1].

## Sources
[S1] NeuDA: Neural Deformable Anchor for High-Fidelity Implicit Surface Reconstruction, https://doi.org/10.1109/cvpr52729.2023.00819, fetched 2026-10-26
