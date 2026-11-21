# CityGS Training Pipeline

Also called: Block-wise Parallel Training with Global Prior, CityGaussian, CityGaussian Training

## Overview
CityGS is a divide-and-conquer training method for large-scale 3D Gaussian Splatting (3DGS) that uses a coarse global prior to enable parallel block-wise finetuning and a Level-of-Detail (LoD) strategy for real-time rendering [S1]. It solves the geometric inaccuracies and floaters caused by independent block training by initializing each block with a globally consistent Gaussian set, ensuring seamless fusion without complex alignment [S1]. Unlike [Momentum-GS](6553-momentum-gs.md), which uses a momentum-updated teacher for self-distillation, CityGS relies on a static coarse prior and spatial contraction for data assignment [S1].

## How it works
**Inputs:**
- A set of input images with known intrinsics $\kappa$ and poses $\tau_i$.
- Initial sparse point cloud (e.g., from COLMAP) [S1].
- Hyperparameters: global training iterations ($N_{global}=30,000$), SSIM threshold $\epsilon$, block bounds $[b_{j,min}, b_{j,max}]$, and LoD compression rates $r_0, r_1, r_2$ [S1].

**Outputs:**
- A fused set of 3D Gaussians $G_{final}$ organized into detail levels for rendering [S1].

**Data Structures:**
Each Gaussian $G_k$ contains:
- Position $p_k \in \mathbb{R}^3$ [S1].
- Opacity $\alpha_k \in [0, 1]$ [S1].
- Covariance defined by scaling and rotation [S1].
- Spherical Harmonics (SH) features $f^k \in \mathbb{R}^{3 \times 16}$ for view-dependent color $c_k \in \mathbb{R}^3$ [S1].

**Steps:**

1. **Generate Global Prior:** Train the initial COLMAP points using all input images for 30,000 iterations with standard 3DGS densification/pruning to produce a coarse global Gaussian set $G_K = \{G_k | k = 1, ..., K\}$ [S1].

2. **Contract Global Prior:** Normalize the positions of $G_K$ to $[-1, 1]$ using the min/max corners $p_{min}, p_{max}$ of the scene bounds to get $\hat{p}_k$. Apply non-linear contraction to handle positions with infinity norm $> 1$:
   $$
   \text{contract}(\hat{p}_k) = 
   \begin{cases} 
   \hat{p}_k & \text{if } \|\hat{p}_k\|_\infty \le 1 \\
   \left(2 - \frac{1}{\|\hat{p}_k\|_\infty}\right) \frac{\hat{p}_k}{\|\hat{p}_k\|_\infty} & \text{if } \|\hat{p}_k\|_\infty > 1
   \end{cases}
   $$
   where $\|\cdot\|_\infty$ is the infinity norm [S1].

3. **Partition Space:** Divide the contracted cubic space evenly into blocks. For each block $j$, identify the subset of Gaussians $G_{K_j}$ contained within the block bounds $[b_{j,min}, b_{j,max}]$ in the contracted space [S1].

4. **Handle Empty Blocks:** If a block $j$ has too few Gaussians ($K_j$ below a threshold), enlarge its bounds $b_{j,min}$ and $b_{j,max}$ until the threshold is met. This adjustment is used only for data assignment to ensure sufficient training data, not for filtering final Gaussians [S1].

5. **Assign Training Data:** For each block $j$ and image pose $\tau_i$, determine if the image should be used for training block $j$ using two criteria combined via logical OR (sum):
   - **Visibility Criterion ($B_1$):** Calculate the SSIM loss between the image rendered by the global prior $I_{G_K}(\tau_i)$ and the image rendered by the block's contribution $I_{G_K \setminus G_{K_j}}(\tau_i)$. If $L_{SSIM}(I_{G_K}(\tau_i), I_{G_K \setminus G_{K_j}}(\tau_i)) > \epsilon$, then $B_1(\tau_i, G_{K_j}) = 1$, else 0 [S1].
   - **Spatial Criterion ($B_2$):** Check if the camera pose position $\hat{p}_{\tau_i}$ (in world coordinates, contracted) falls within the block bounds:
     $$
     B_2(\tau_i, G_{K_j}) = 
     \begin{cases} 
     1 & \text{if } b_{j,min} \le \text{contract}(\hat{p}_{\tau_i}) < b_{j,max} \\
     0 & \text{otherwise}
     \end{cases}
     $$
     [S1].
   - **Final Assignment:** $B(\tau_i, G_{K_j}) = B_1(\tau_i, G_{K_j}) + B_2(\tau_i, G_{K_j})$. If $B > 0$, the image is assigned to block $j$ [S1].

6. **Parallel Finetuning:** Train each block $j$ in parallel. Initialize the block's Gaussians with the corresponding subset $G_{K_j}$ from the global prior. Train in the **original uncontracted space** using the assigned images. The loss is the weighted sum of L1 and SSIM losses [S1].

7. **Post-processing:** After training, filter the finetuned Gaussians of block $j$ to keep only those physically located within the block's original spatial bounds (in uncontracted space). Concatenate all filtered blocks to form the final dense model [S1].

8. **Level-of-Detail (LoD) Generation:** Compress the fused Gaussians into multiple detail levels using compression rates $r_0, r_1, r_2$ [S1].

9. **Rendering:** For a given view, dynamically select the appropriate LoD level based on camera distance or resolution requirements. Splat the selected Gaussians to screen space, sort by depth, and render via alpha blending:
   $$
   c_i(\mathbf{x}) = \sum_{k=1}^K \alpha_k \mathbf{c}_k G_k^{2D}(\mathbf{x}) \prod_{t=1}^{k-1} (1 - \alpha_t G_t^{2D}(\mathbf{x}))
   $$
   where $G_k^{2D} = \text{proj}(G_k, \kappa, \tau_i)$ is the projected 2D Gaussian distribution [S1].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def citygs_training_pipeline(images, poses, initial_points):
    # Step 1: Generate Global Prior
    # Train for 30,000 iterations using all images
    global_gaussians = train_3dgs(initial_points, images, poses, iterations=30000)
    
    # Step 2: Contract Global Prior
    p_min, p_max = get_scene_bounds(global_gaussians.positions)
    contracted_gaussians = []
    for g in global_gaussians:
        p_hat = normalize(g.position, p_min, p_max)
        if np.linalg.norm(p_hat, ord=np.inf) <= 1:
            contracted_pos = p_hat
        else:
            contracted_pos = (2 - 1/np.linalg.norm(p_hat, ord=np.inf)) * (p_hat / np.linalg.norm(p_hat, ord=np.inf))
        contracted_gaussians.append(Gaussian(position=contracted_pos, attributes=g.attributes))
    
    # Step 3: Partition Space
    blocks = create_even_partitions(contracted_gaussians)
    
    # Step 4 & 5: Assign Data and Handle Empty Blocks
    block_data_assignments = {}
    for j, block in enumerate(blocks):
        # Identify Gaussians in block bounds
        g_kj = [g for g in contracted_gaussians if is_in_bounds(g.position, block.min, block.max)]
        
        # Handle empty blocks by enlarging bounds if necessary
        if len(g_kj) < threshold:
            block.min, block.max = enlarge_bounds(block.min, block.max, g_kj, threshold)
            g_kj = [g for g in contracted_gaussians if is_in_bounds(g.position, block.min, block.max)]
        
        assigned_images = []
        for i, (img, pose) in enumerate(zip(images, poses)):
            # B1: Visibility check
            # Render global prior and block contribution
            img_global = render(global_gaussians, pose)
            img_block_contrib = render(g_kj, pose) # Simplified: render difference
            ssim_loss = compute_ssim(img_global, img_block_contrib)
            b1 = 1 if ssim_loss > epsilon else 0
            
            # B2: Spatial check
            pose_pos_contracted = contract(pose.position, p_min, p_max)
            b2 = 1 if is_in_bounds(pose_pos_contracted, block.min, block.max) else 0
            
            if b1 + b2 > 0:
                assigned_images.append(img)
        
        block_data_assignments[j] = (g_kj, assigned_images)
    
    # Step 6: Parallel Finetuning
    finetuned_blocks = []
    for j, (g_kj, assigned_images) in block_data_assignments.items():
        # Initialize with global prior subset
        block_gaussians = copy(g_kj)
        # Train in original uncontracted space
        block_gaussians = finetune(block_gaussians, assigned_images, loss='l1+ssim')
        
        # Step 7: Post-processing
        # Filter to keep only Gaussians within original spatial bounds (uncontracted)
        # Note: block bounds were defined in contracted space, need to map back or use original bounds
        # The source says "filter out the finetuned Gaussians contained within its spatial bound"
        # Assuming spatial bound refers to the block's region in world space
        filtered_gaussians = [g for g in block_gaussians if is_in_world_bounds(g.position, block.world_min, block.world_max)]
        finetuned_blocks.append(filtered_gaussians)
    
    # Concatenate blocks
    final_gaussians = concatenate(finetuned_blocks)
    
    # Step 8: LoD Generation
    lod_levels = generate_lod(final_gaussians, rates=[r0, r1, r2])
    
    return lod_levels

def contract(p_hat, p_min, p_max):
    # Normalize
    p_norm = (p_hat - p_min) / (p_max - p_min) * 2 - 1 # Map to [-1, 1]
    # This matches the description: "normalize positions to [-1, 1]"
    # Then apply contraction
    norm_inf = np.linalg.norm(p_norm, ord=np.inf)
    if norm_inf <= 1:
        return p_norm
    else:
        return (2 - 1/norm_inf) * (p_norm / norm_inf)
```

**Edge Cases and Pitfalls:**
- **Empty Blocks:** Extremely uneven Gaussian distributions may result in blocks with no Gaussians. The algorithm explicitly enlarges block bounds to ensure a minimum number of Gaussians, preventing overfitting due to lack of training data [S1].
- **Contraction Boundary:** The contraction function is continuous but non-linear. Positions with infinity norm exactly 1 are unaffected [S1].
- **SSIM Threshold:** The choice of $\epsilon$ affects data assignment. Too high, and blocks may lack training data; too low, and blocks may train on irrelevant views [S1].

## Operating characteristics
- **Time Complexity:** Training is parallelized across blocks, reducing wall-clock time compared to single-process global training. The global prior generation takes 30,000 iterations [S1].
- **Memory Complexity:** Requires storing the global prior and all block data. The contraction step is memory-efficient as it operates on positions [S1].
- **Performance:** Achieves real-time rendering on large-scale scenes using LoD [S1].
- **Strengths:** Eliminates floaters and geometric inaccuracies at block boundaries; enables parallel training; simple fusion via concatenation [S1].
- **Limitations:** Requires a high-quality global prior; performance depends on the effectiveness of the contraction and partitioning [S1].

## Background
CityGaussian (CityGS) was introduced to address the challenges of training and rendering large-scale 3DGS in real-time [S1]. It builds on the 3DGS method [S1] and introduces a divide-and-conquer strategy with a global prior and LoD [S1].

## Sources
[S1] CityGaussian: Real-Time High-Quality Large-Scale Scene Rendering with Gaussians, https://doi.org/10.1007/978-3-031-72640-8_15, fetched 2026-10-26
