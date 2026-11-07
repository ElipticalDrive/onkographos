# Neural Light Field (NeLF) via R2L Distillation

Also called: R2L, NeLF via R2L Distillation

## Overview
R2L is a knowledge distillation method that trains a Neural Light Field (NeLF) student model to approximate a pre-trained Neural Radiance Field (NeRF) teacher model [S1]. Unlike NeRF, which requires ray marching and alpha-composition to compute pixel color, R2L maps a 4D oriented ray directly to 3D RGB color in a single forward pass [S1]. This bypasses the iterative sampling bottleneck of NeRF, achieving approximately 30× wall-time speedup while maintaining comparable rendering quality [S1]. It builds on [Neural Light Field Representations](9273-neural-light-field-representations.md) by providing a specific training procedure and architecture for efficient NeLF instantiation.

## How it works
**Inputs and Outputs:**
*   **Input:** A 4D oriented ray, parameterized by origin $(x_o, y_o, z_o)$ and normalized direction $(x_d, y_d, z_d)$.
*   **Output:** 3D RGB color $(r, g, b)$.

**Data Structures:**
The NeLF network $G_\phi$ is a Deep Residual MLP. The input representation is a concatenated vector of spatial coordinates of $K$ sampled points along the ray, forming a $3K$-dimensional vector [S1].
*   **Network Configurations:**
    *   High Quality: $W256D88$ (Width=256, Depth=88), ~12M FLOPs per ray [S1].
    *   Efficient: $W181D88$, $W256D44$, $W363D22$, ~6M FLOPs per ray [S1].
*   **Positional Encoding:** Input coordinates are enriched using positional encoding following [NeRF](52-nerf.md) [S1, S53].

**Pseudo Data Synthesis:**
Since NeLF requires massive data to train deep networks, pseudo data is synthesized from the pre-trained NeRF teacher $F_{\Theta^*}$ [S1].
1.  Infer a viewing bounding box from the training camera poses [S1].
2.  Sample ray origins $(x_o, y_o, z_o)$ and normalized directions $(x_d, y_d, z_d)$ uniformly from this box:
    $$
    \begin{aligned}
    x_o &\sim U(x_{o,\min}, x_{o,\max}), & x_d &\sim U(x_{d,\min}, x_{d,\max}) \\
    y_o &\sim U(y_{o,\min}, y_{o,\max}), & y_d &\sim U(y_{d,\min}, y_{d,\max}) \\
    z_o &\sim U(z_{o,\min}, z_{o,\max}), & z_d &\sim U(z_{d,\min}, z_{d,\max})
    \end{aligned}
    $$
3.  Query the teacher NeRF $F_{\Theta^*}$ with the 6D input to obtain target RGB $(\hat{r}, \hat{g}, \hat{b})$ [S1]:
    $$
    (\hat{r}, \hat{g}, \hat{b}) = F_{\Theta^*}(x_o, y_o, z_o, x_d, y_d, z_d)
    $$
4.  Store the tuple $(x_o, y_o, z_o, x_d, y_d, z_d, \hat{r}, \hat{g}, \hat{b})$ as a training sample [S1].
5.  Synthesize 10,000 images (pseudo data) [S1].

**Training Procedure:**
1.  **Ray Representation:** For each ray in the batch, sample $K$ points along the ray.
    *   **Training:** Use stratified sampling to prevent overfitting to specific point locations [S1].
    *   **Testing:** Use evenly spaced points [S1].
    *   $K=16$ is used [S1].
2.  **Loss Function:** Minimize Mean Squared Error (MSE) between the NeLF output and the teacher's target RGB [S1]:
    $$
    \mathcal{L} = \text{MSE}(G_\phi(x_o, y_o, z_o, x_d, y_d, z_d), (\hat{r}, \hat{g}, \hat{b}))
    $$
3.  **Hard Example Mining:**
    *   Maintain a pool of hard examples (samples with larger loss values) [S1].
    *   In each iteration, sort batch losses in ascending order.
    *   Add the top $r\%$ (pre-defined percentage constant) to the pool [S1].
    *   Randomly pick $r\%$ of hard examples from the pool to augment the training batch [S1].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import numpy as np

def synthesize_pseudo_data(teacher_nerf, bounding_box, num_samples=10000):
    """
    Synthesize pseudo data by querying the pre-trained NeRF teacher.
    bounding_box: dict with keys xmin_o, xmax_o, ymin_o, ymax_o, zmin_o, zmax_o,
                  xmin_d, xmax_d, ymin_d, ymax_d, zmin_d, zmax_d
    """
    data = []
    for _ in range(num_samples):
        # Sample origin uniformly within bounding box
        xo = np.random.uniform(bounding_box['xmin_o'], bounding_box['xmax_o'])
        yo = np.random.uniform(bounding_box['ymin_o'], bounding_box['ymax_o'])
        zo = np.random.uniform(bounding_box['zmin_o'], bounding_box['zmax_o'])
        
        # Sample direction uniformly within bounding box
        xd = np.random.uniform(bounding_box['xmin_d'], bounding_box['xmax_d'])
        yd = np.random.uniform(bounding_box['ymin_d'], bounding_box['ymax_d'])
        zd = np.random.uniform(bounding_box['zmin_d'], bounding_box['zmax_d'])
        
        # Normalize direction to unit vector
        norm = np.sqrt(xd**2 + yd**2 + zd**2)
        xd, yd, zd = xd/norm, yd/norm, zd/norm
        
        # Query teacher NeRF
        # Note: Teacher expects 5D input (x,y,z, theta, phi) or similar.
        # The source [S1] implies querying with (xo, yo, zo, xd, yd, zd) directly
        # to get RGB. We assume the teacher handles the conversion or the 
        # source notation F(xo,yo,zo,xd,yd,zd) is a simplified representation 
        # of the ray query.
        target_rgb = teacher_nerf.query(xo, yo, zo, xd, yd, zd)
        
        data.append({
            'origin': (xo, yo, zo),
            'direction': (xd, yd, zd),
            'target_rgb': target_rgb
        })
    return data

def stratified_sample_points(ray_origin, ray_direction, K=16, near=0.0, far=1.0):
    """
    Sample K points along a ray using stratified sampling.
    """
    # Generate K random offsets
    offsets = np.random.uniform(0, 1, K)
    # Stratified sampling: divide [0,1] into K intervals and pick one point per interval
    # This ensures coverage across the ray segment
    stratified_offsets = np.sort(offsets) # Simplified stratification
    # More precise stratified sampling:
    # intervals = np.linspace(0, 1, K+1)
    # offsets = np.random.uniform(intervals[:-1], intervals[1:])
    # offsets.sort()
    
    # Map offsets to positions along the ray
    # t values between near and far
    t_values = near + (far - near) * stratified_offsets
    
    points = ray_origin + t_values[:, np.newaxis] * ray_direction
    return points

def train_r2l(student_nerf, pseudo_data, hard_example_pool, r_percent=0.2, 
              batch_size=98304, num_iterations=1000000, lr=5e-4):
    """
    Train the R2L NeLF model.
    """
    optimizer = Adam(student_nerf.parameters(), lr=lr)
    
    for iteration in range(num_iterations):
        # 1. Sample a batch of rays from pseudo data
        batch_indices = np.random.choice(len(pseudo_data), batch_size, replace=False)
        batch_samples = [pseudo_data[i] for i in batch_indices]
        
        # 2. Prepare inputs
        inputs = []
        targets = []
        
        for sample in batch_samples:
            origin = np.array(sample['origin'])
            direction = np.array(sample['direction'])
            
            # Stratified sampling for training
            points = stratified_sample_points(origin, direction, K=16)
            
            # Flatten points to 3K vector
            input_vec = points.flatten()
            
            # Apply positional encoding (simplified placeholder)
            input_vec = apply_positional_encoding(input_vec)
            
            inputs.append(input_vec)
            targets.append(sample['target_rgb'])
            
        inputs = np.array(inputs)
        targets = np.array(targets)
        
        # 3. Forward pass
        predictions = student_nerf(inputs)
        
        # 4. Compute Loss
        loss = np.mean((predictions - targets) ** 2)
        
        # 5. Hard Example Mining
        # Calculate per-sample loss
        per_sample_loss = np.mean((predictions - targets) ** 2, axis=1)
        
        # Sort losses
        sorted_indices = np.argsort(per_sample_loss)
        top_r_count = int(batch_size * r_percent)
        
        # Add top r% to pool
        hard_examples_from_batch = [batch_samples[i] for i in sorted_indices[-top_r_count:]]
        hard_example_pool.extend(hard_examples_from_batch)
        
        # Pick r% from pool to augment batch (replace easy examples)
        # For simplicity, we assume we replace the last r% of the current batch
        # with random samples from the pool
        pool_size = len(hard_example_pool)
        if pool_size > 0:
            pool_indices = np.random.choice(pool_size, top_r_count, replace=False)
            hard_examples = [hard_example_pool[i] for i in pool_indices]
            
            # Replace the last top_r_count samples in the batch
            for i, hard_ex in enumerate(hard_examples):
                # Re-process hard example
                origin = np.array(hard_ex['origin'])
                direction = np.array(hard_ex['direction'])
                points = stratified_sample_points(origin, direction, K=16)
                input_vec = points.flatten()
                input_vec = apply_positional_encoding(input_vec)
                
                inputs[batch_size - top_r_count + i] = input_vec
                targets[batch_size - top_r_count + i] = hard_ex['target_rgb']
                
            # Recompute loss and predictions for the augmented batch
            predictions = student_nerf(inputs)
            loss = np.mean((predictions - targets) ** 2)
            
        # 6. Backward pass
        optimizer.zero_grad()
        loss.backward()
        optimizer.step()
        
    return student_nerf
```

**Edge Cases and Pitfalls:**
*   **Network Depth:** Deep networks for NeRF/NeLF suffer from diminishing returns if not designed carefully. Residual connections are critical to enable greater depth [S1].
*   **Generalization:** Randomly sampling ray origins/directions may lead to trivial parts of the scene (e.g., white background). Hard example mining mitigates this by focusing on high-frequency details [S1].
*   **Input Representation:** Concatenated coordinates perform better than Plücker coordinates for this specific distillation setup (PSNR 29.50 vs 29.08 on Lego scene) [S1].

## Operating characteristics
*   **Speed:** Achieves ~30× wall-time speedup over baseline NeRF [S1]. This is significantly faster than other NeLF methods like [R2L (Neural Light Field)](8670-r2l-neural-light-field.md) which achieve ~5× speedup [S1].
*   **Quality:** Comparable to NeRF. On the Lego scene with W181D88 network trained on pseudo data for 200K iterations, it achieves PSNR 29.50 [S1].
*   **Memory:** Storage is comparable to NeRF (e.g., 2.4 MB for Teacher NeRF vs 23.7 MB for R2L with pseudo data in some configurations, though table values vary by method comparison) [S1]. The FLOPs are fixed per ray (e.g., 12M for W256D88) [S1].
*   **Training Time:** Requires 1,000k iterations with a batch size of 98,304 rays [S1].
*   **Strengths:** Single forward pass per ray; no ray marching; high speedup; effective for complex real-world scenes [S1].
*   **Limitations:** Requires a pre-trained NeRF teacher; training is data-hungry (requires 10k synthesized images); deep network training is sensitive to architecture (requires residual blocks) [S1].

## Background
R2L was proposed by Wang et al. in the paper "R2L: Distilling Neural Radiance Field to Neural Light Field for Efficient Novel View Synthesis" [S1]. It addresses the efficiency bottleneck of NeRF, which requires multiple forward passes per ray due to ray marching [S1]. The method leverages knowledge distillation, a technique pioneered by Buciluǎ et al. [8] and refined by Hinton et al. [18], to transfer knowledge from a large teacher model to a smaller, faster student model [S1]. It is the first method to address NeRF efficiency through network architecture optimization (deep residual MLP) rather than just inheriting NeRF's MLP structure [S1].

## Sources
[S1] R2L: Distilling Neural Radiance Field to Neural Light Field for Efficient Novel View Synthesis, https://doi.org/10.1007/978-3-031-19821-2_35, fetched 2026-10-26
