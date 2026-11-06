# Probability-based Ray Sampling with Quadtree Subdivision

Also called: Adaptive Quadtree Subdivision, Fast Learning Radiance Fields by Shooting Much Fewer Rays method, Ours, Quadtree Subdivision

## Overview
This method accelerates radiance field training by dynamically adjusting ray distribution through context-based probability sampling and adaptive quadtree subdivision [S1]. It reduces training time by 15–40% while maintaining or improving accuracy compared to baseline methods like NeRF, Plenoxels, and Instant-NGP [S1]. Unlike uniform sampling or fixed-block approaches (e.g., iMap), it subdivides image regions based on rendering error, allowing complex areas to receive more rays while marking converged simple regions to reduce computational load [S1].

## How it works
**Inputs and Outputs**
- **Inputs**: Input views (images) with dimensions $H_i \times W_i$; a radiance field model $\Theta$ (e.g., NeRF, Plenoxels); a convergence threshold $\alpha$; a constant ray count for marked nodes $n_0$; a random sampling ratio $\rho$ [S1].
- **Outputs**: A trained radiance field; a quadtree structure per view indicating which leaf nodes are marked (converged) or unmarked (active) [S1].

**Data Structures**
- **Quadtree**: Constructed on each input view. Each node represents a rectangular block of the image.
    - **Leaf Node**: Represents a block of pixels.
    - **State**: Each leaf node has a state: `unmarked` (active) or `marked` (converged).
    - **Subdivision**: An unmarked leaf node can be subdivided into four equal-sized child leaf nodes.
- **Probability Map**: A 2D array of the same dimensions as the input image, storing the sampling probability density for each pixel [S1].

**Numbered Steps**

1.  **Initialization**:
    - For each input view $I_i$ with height $H_i$ and width $W_i$, construct a quadtree with a single root node covering the entire image. Initially, all nodes are unmarked.
    - Calculate the initial number of rays $M_i = H_i \times W_i$ [S1].
    - Compute the initial probability density for each pixel $(u, v)$ using the standard deviation of color in a $3 \times 3$ patch [S1]:
      $$
      \text{variance}(c(u, v)) = \frac{1}{9} \sum_{x \in \{u-1, u, u+1\}} \sum_{y \in \{v-1, v, v+1\}} [c(x, y) - \bar{c}]^2
      $$
      where $\bar{c}$ is the mean color in the patch. Normalize these variances to form a probability distribution $g'(u, v)$ [S1].

2.  **Ray Sampling**:
    - For each active (unmarked) leaf node $F$ in the quadtree:
        - Determine the set of pixels $P_F$ covered by $F$.
        - Sample rays for pixels in $P_F$ based on the probability distribution $g'(u, v)$ restricted to $P_F$.
        - Add a random sampling component with ratio $\rho$ to ensure trivial areas (smooth surfaces) are also learned [S1].
    - For each marked leaf node $F$:
        - Sample a fixed small number of rays $n_0$ (typically $n_0 = 10$) randomly or uniformly within $F$ [S1].

3.  **Rendering and Error Calculation**:
    - Shoot the sampled rays through the radiance field model $\Theta$ to render colors $\hat{c}$.
    - For each leaf node $F$, calculate the average rendering error $e_F$ over all emitted rays $r_i \in F$:
      $$
      e_F = \text{mean}_{r_i \in F} (V_\Theta(r_i) - c_i)
      $$
      where $c_i$ is the ground truth color of the pixel emitting ray $r_i$, and $V_\Theta(r_i)$ is the rendered color [S1]. Note: The source formula uses a linear difference; in practice, this may be squared error or L1, but the text specifies $V_\Theta(r_i) - c_i$ [S1].

4.  **Quadtree Subdivision and Marking**:
    - For each leaf node $F$:
        - If $e_F < \alpha$ (threshold): Mark $F$ as `marked` (converged). Do not subdivide further.
        - If $e_F \ge \alpha$: Mark $F$ as `unmarked` (if not already) and subdivide it into four smaller leaf nodes. These new children are initially unmarked [S1].

5.  **Iteration**:
    - Repeat steps 2–4 for subsequent training iterations. The total number of rays $M_i$ decreases as more nodes become marked [S1].

**Pseudocode**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def initialize_quadtree(image_height, image_width):
    """
    Creates a quadtree with a single root node covering the entire image.
    Returns the root node.
    """
    root = Node(x=0, y=0, w=image_width, h=image_height, state='unmarked')
    return root

def calculate_probability_map(image):
    """
    Calculates probability density for each pixel using the standard deviation 
    of color in a 3x3 patch.
    """
    h, w, _ = image.shape
    prob_map = np.zeros((h, w))
    
    # Pad image for boundary handling
    padded = np.pad(image, pad_width=1, mode='edge')
    
    for u in range(h):
        for v in range(w):
            # Extract 3x3 patch centered at (u, v)
            # Note: indices in padded array are shifted by 1
            patch = padded[u : u+3, v : v+3, :]
            mean_color = np.mean(patch, axis=(0, 1))
            variance = np.mean((patch - mean_color) ** 2)
            prob_map[u, v] = variance
            
    # Normalize to sum to 1
    prob_map /= np.sum(prob_map)
    return prob_map

def sample_rays_from_node(node, prob_map, n0, random_ratio):
    """
    Samples rays for a specific leaf node.
    If marked, samples n0 rays.
    If unmarked, samples rays based on probability map and random ratio.
    """
    if node.state == 'marked':
        # Sample n0 rays randomly within the node's bounds
        rays = []
        for _ in range(n0):
            x = random.uniform(node.x, node.x + node.w)
            y = random.uniform(node.y, node.y + node.h)
            rays.append((x, y))
        return rays
    
    else:
        # Unmarked node: sample based on probability distribution
        # Extract probability sub-map for this node
        h_node = int(node.h)
        w_node = int(node.w)
        x_start = int(node.x)
        y_start = int(node.y)
        
        sub_prob = prob_map[x_start:x_start+h_node, y_start:y_start+w_node]
        
        # Normalize sub-prob
        sub_prob_sum = np.sum(sub_prob)
        if sub_prob_sum > 0:
            sub_prob /= sub_prob_sum
        else:
            sub_prob = np.ones_like(sub_prob) / (h_node * w_node)
            
        # Flatten for sampling
        flat_prob = sub_prob.flatten()
        num_pixels = h_node * w_node
        
        # Determine number of rays for this node (equal to pixel count for unmarked)
        num_rays = num_pixels
        
        # Sample indices based on probability
        indices = np.random.choice(num_pixels, size=num_rays, p=flat_prob)
        
        # Convert indices to coordinates
        rays = []
        for idx in indices:
            row = idx // w_node
            col = idx % w_node
            x = x_start + col + random.uniform(0, 1)
            y = y_start + row + random.uniform(0, 1)
            rays.append((x, y))
            
        # Add random sampling to cover trivial areas
        num_random = int(num_rays * random_ratio)
        for _ in range(num_random):
            x = random.uniform(node.x, node.x + node.w)
            y = random.uniform(node.y, node.y + node.h)
            rays.append((x, y))
            
        return rays

def calculate_node_error(node, rays, model, ground_truth_image):
    """
    Calculates average rendering error for rays in a node.
    """
    errors = []
    for (x, y) in rays:
        # Render color for ray at (x, y)
        rendered_color = model.render_ray(x, y)
        # Get ground truth color
        gt_color = ground_truth_image[int(y), int(x)]
        # Calculate error (mean of difference)
        error = np.mean(rendered_color - gt_color)
        errors.append(error)
        
    if not errors:
        return 0.0
    return np.mean(errors)

def train_with_quadtree(image, model, threshold_alpha, n0, random_ratio, max_subdivisions):
    """
    Main training loop with adaptive quadtree subdivision.
    """
    h, w, _ = image.shape
    root = initialize_quadtree(h, w)
    prob_map = calculate_probability_map(image)
    
    # Flatten list of all leaf nodes for traversal
    leaf_nodes = [root]
    
    for iteration in range(max_subdivisions):
        current_leaf_nodes = []
        
        # 1. Sample rays and render
        all_rays = []
        for node in leaf_nodes:
            rays = sample_rays_from_node(node, prob_map, n0, random_ratio)
            all_rays.extend(rays)
            
        # 2. Train model on all_rays (simplified)
        # In practice, this involves gradient descent steps
        model.train_step(all_rays, image)
        
        # 3. Calculate errors and update quadtree
        next_leaf_nodes = []
        for node in leaf_nodes:
            # Re-sample rays for error calculation if necessary, or use previous rays
            # For simplicity, assume we re-sample or track rays per node
            rays_for_node = sample_rays_from_node(node, prob_map, n0, random_ratio)
            error = calculate_node_error(node, rays_for_node, model, image)
            
            if error < threshold_alpha:
                node.state = 'marked'
                # Marked nodes are not subdivided
                next_leaf_nodes.append(node)
            else:
                node.state = 'unmarked'
                # Subdivide into 4 children
                half_w = node.w / 2
                half_h = node.h / 2
                x, y = node.x, node.y
                
                # Top-left
                child1 = Node(x, y, half_w, half_h, 'unmarked')
                # Top-right
                child2 = Node(x + half_w, y, half_w, half_h, 'unmarked')
                # Bottom-left
                child3 = Node(x, y + half_h, half_w, half_h, 'unmarked')
                # Bottom-right
                child4 = Node(x + half_w, y + half_h, half_w, half_h, 'unmarked')
                
                node.children = [child1, child2, child3, child4]
                next_leaf_nodes.extend([child1, child2, child3, child4])
                
        leaf_nodes = next_leaf_nodes
        
        # Check if all nodes are marked (converged)
        if all(node.state == 'marked' for node in leaf_nodes):
            break
            
    return model, root
```

**Parameters**
- $\alpha$: Convergence threshold. If average rendering error $e_F < \alpha$, the node is marked. Value not explicitly given in sources, but discussed in ablation study [S1].
- $n_0$: Constant number of rays sampled on marked leaf nodes. Typically set to $n_0 = 10$ [S1].
- $\rho$: Random sampling ratio. Helps learn trivial areas. Optimal value found via ablation (peak PSNR around 20-40% ratio, but exact optimal not specified, just that it helps) [S1].
- Patch Size: $3 \times 3$ is used. Alternatives ($5 \times 5, 7 \times 7, 9 \times 9$) and metrics (variation, entropy) were tested but found insensitive [S1].

**Edge Cases and Pitfalls**
- **Precision**: The error calculation $e_F$ uses mean difference. If rendered colors and ground truth are in different ranges, this may need normalization [S1].
- **Boundaries**: When calculating variance for pixels on image boundaries, padding (e.g., edge padding) is required to maintain the $3 \times 3$ patch [S1].
- **Degenerate Input**: If an image is uniform (zero variance), the probability map will be zero everywhere. Normalization handles this by distributing probability uniformly or requiring a fallback [S1].

## Subtopics

### [Training Ray Marching and Integration](2199-training-ray-marching-and-integration.md)
Defines how rays are integrated through the volume to produce colors, which is the core rendering step used to calculate errors in this method.

### [Coarse-to-Fine NeRF](3406-coarse-to-fine-nerf.md)
A related optimization strategy that uses multiple resolutions, whereas this method uses spatial subdivision of the image plane.

### [Mip-NeRF 360](3306-mip-nerf-360.md)
A baseline method that this framework can accelerate, focusing on anti-aliasing rather than adaptive ray count.

## Operating characteristics
- **Time Complexity**: Training time is reduced by 15–40% compared to baselines (NeRF, Plenoxels, Instant-NGP) [S1]. The reduction increases as training progresses because more nodes become marked and require fewer rays [S1].
- **Memory Complexity**: The quadtree structure adds minimal overhead. The number of rays $M_i$ decreases over time, reducing memory for ray buffers [S1].
- **Measured Performance**:
    - **LLFF (Forward-Facing)**: Training time reduced from 66.6 min (Plenoxels) to 40.4 min (39% reduction), PSNR increased from 24.60 to 25.12 [S1].
    - **Tanks and Temples**: Instant-NGP training time reduced from 354s to 285s (19% reduction), PSNR increased from 27.91 to 28.25 [S1].
    - **DTU**: NeuS training time reduced from 9.2h to 7.3h (21% reduction), Chamfer Distance improved from 0.87 to 0.73 [S1].
- **Strengths**: General framework applicable to various radiance field methods (NeRF, NeRF++, Plenoxels, Instant-NGP, Mip-NeRF 360, NeuS). Balances efficiency and effectiveness, improving early-stage training speed and later-stage efficiency [S1].
- **Limitations**: Initial overhead for sampling and subdividing may cause a tiny increase in training time at the very beginning [S1]. Performance depends on the choice of threshold $\alpha$ and random ratio $\rho$ [S1].

## Background
- **Origin**: Proposed in "Fast Learning Radiance Fields by Shooting Much Fewer Rays" (2023) [S1].
- **History**: The method addresses the high computational cost of training radiance fields by reducing the number of rays needed. It differs from iMap by using error-based subdivision rather than point-count distribution and by allowing the number of leaf nodes to change during training [S1].
- **Usage**: Applied as a general acceleration framework for NeRF, NeRF++, Plenoxels, Instant-NGP, Mip-NeRF 360, and NeuS [S1].

## Sources
[S1] Fast Learning Radiance Fields by Shooting Much Fewer Rays, https://doi.org/10.1109/tip.2023.3267049, fetched 2026-10-26
