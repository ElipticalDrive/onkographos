# Probability-based Ray Sampling with Quadtree Subdivision

Also called: Adaptive Quadtree Subdivision, Fast Learning Radiance Fields by Shooting Much Fewer Rays method, Ours, Quadtree Subdivision

## Overview
This method accelerates radiance field training by dynamically adjusting ray distribution through context-based probability sampling and adaptive quadtree subdivision [S1]. It reduces training time by 15–40% while maintaining or improving reconstruction accuracy compared to baseline methods like NeRF, Plenoxels, and Instant-NGP [S1]. Unlike sibling methods that use fixed block sizes or uniform sampling within blocks, this approach subdivides image regions into variable-sized leaf nodes based on rendering error, allowing for finer resolution in complex areas and coarser sampling in converged regions [S1].

## How it works
The method operates on a per-view basis, constructing a spatial hierarchy to guide ray sampling. It combines a prior probability distribution derived from image context with an error-driven subdivision mechanism.

### Inputs and Outputs
*   **Inputs:** Input views (images) with dimensions $H_i \times W_i$; ground truth colors $c_i$; a radiance field model $\Theta$ (e.g., NeRF, Plenoxels, NeuS); a convergence threshold $\alpha$; a constant ray count for marked nodes $n_0$.
*   **Outputs:** A set of sampled rays for training the radiance field; the final rendered images.

### Data Structures
**Quadtree Node:**
Each node in the quadtree represents a rectangular block of the image.
*   `is_leaf`: Boolean, true if the node has no children.
*   `is_marked`: Boolean, true if the node has converged and should not be subdivided.
*   `x, y`: Top-left pixel coordinates of the block.
*   `width, height`: Dimensions of the block in pixels.
*   `children`: List of 4 child nodes (N, S, E, W) if not a leaf.

**Probability Map:**
A 2D array of size $H_i \times W_i$ storing the prior probability density for each pixel.

### Numbered Steps

1.  **Initialization:**
    For each input view $I_i$ with height $H_i$ and width $W_i$:
    *   Construct a root quadtree node covering the entire image ($H_i \times W_i$).
    *   Initialize the root as a leaf node.
    *   Mark all nodes as unmarked.
    *   Calculate the initial number of rays $M_i = H_i \times W_i$.

2.  **Probability Distribution Calculation:**
    For each pixel $(u, v)$ in the image:
    *   Define a 3x3 patch centered at $(u, v)$, where $x \in \{u-1, u, u+1\}$ and $y \in \{v-1, v, v+1\}$ [S1].
    *   Calculate the mean color $\bar{c}$ of the patch:
        $$ \bar{c} = \frac{1}{9} \sum_{x,y} c(x, y) $$
    *   Calculate the variance (standard deviation squared) of the color in the patch:
        $$ \text{variance}(c(u, v)) = \frac{1}{9} \sum_{x,y} [c(x, y) - \bar{c}]^2 $$
    *   Normalize the variances across the image to form a probability density function $g'(u, v)$ [S1].
    *   *Note:* The sources state that patch sizes (5x5, 7x7, 9x9) and other metrics (variation, entropy) were tested but found insensitive; the 3x3 standard deviation is used [S1].

3.  **Ray Sampling and Rendering:**
    For each leaf node $F$ in the quadtree:
    *   **If $F$ is marked:**
        *   Sample $n_0$ rays uniformly or randomly within the bounds of $F$. The sources specify $n_0 = 10$ as a typical value [S1].
    *   **If $F$ is unmarked:**
        *   Determine the number of rays to sample. The sources imply that for unmarked nodes, the number of sampled rays is equal to the number of pixels in the node (i.e., one ray per pixel of the block resolution) [S1].
        *   For each target pixel position within $F$, select the pixel coordinate $(u, v)$ based on the prior probability distribution $g'(u, v)$ calculated in Step 2.
        *   Add a random sampling ratio to the selection to ensure trivial/smooth areas are also learned [S1].
        *   Shoot rays through the selected pixels into the scene.
        *   Render the color $\hat{c}_i$ for each ray using the volume rendering equation [S1]:
            $$ \hat{c}_i = \sum_{j=1}^{N} T_j (1 - \exp(-\sigma_j \delta_j)) c_j $$
            where $T_j = \exp(-\sum_{t=1}^{j-1} \sigma_t \delta_t)$ [S1].

4.  **Error Calculation:**
    For each leaf node $F$:
    *   Calculate the average rendering error $e_F$ over all rays $r_i$ emitted from pixels in $F$:
        $$ e_F = \text{mean}_{r_i \in F} (\|\hat{c}_i - c_i\|) $$
        where $c_i$ is the ground truth color of the pixel emitting ray $r_i$ [S1]. The sources use the mean of the difference, implying an L1 or L2 norm; the formula notation $V_\Theta(r_i) - c_i$ suggests a vector difference, typically handled by an L2 loss in practice, but the text says "mean... (V - c)" [S1].

5.  **Subdivision and Marking:**
    For each leaf node $F$:
    *   **If $e_F < \alpha$:**
        *   Mark node $F$ as converged (`is_marked = true`).
        *   Do not subdivide.
    *   **If $e_F \ge \alpha$:**
        *   Subdivide node $F$ into four smaller leaf nodes (children).
        *   Reset `is_marked` to false for the new children.
        *   The threshold $\alpha$ is a pre-defined constant [S1].

6.  **Iteration:**
    *   Repeat Steps 3–5 for subsequent training iterations or epochs.
    *   The total number of rays $M_{il}$ after $l$ subdivisions is given by:
        $$ M_{il} = Q_{l1} \times \frac{H_i \times W_i}{2^l} + Q_{l2} \times n_0 $$
        where $Q_{l1}$ is the number of unmarked leaf nodes and $Q_{l2}$ is the number of marked leaf nodes at subdivision level $l$ [S1].

### Pseudocode

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def probability_based_quadtree_sampling(image, ground_truth, model, threshold_alpha, n0=10):
    """
    Performs adaptive ray sampling using quadtree subdivision.
    
    Args:
        image: Input image array of shape (H, W, 3).
        ground_truth: Ground truth color array of shape (H, W, 3).
        model: Radiance field model with a render_ray function.
        threshold_alpha: Convergence threshold for error.
        n0: Number of rays to sample in marked (converged) nodes.
        
    Returns:
        rays: List of ray origins and directions.
        targets: List of target colors.
    """
    H, W, _ = image.shape
    
    # Step 2: Calculate Probability Distribution
    # Use 3x3 patch standard deviation
    # Pad image for boundary handling
    padded_image = np.pad(image, ((1, 1), (1, 1), (0, 0)), mode='edge')
    
    prob_map = np.zeros((H, W))
    for u in range(H):
        for v in range(W):
            # Extract 3x3 patch centered at u, v
            # Note: padded indices are u+1, v+1
            patch = padded_image[u : u+3, v : v+3, :]
            mean_c = np.mean(patch, axis=(0, 1))
            variance = np.mean((patch - mean_c)**2)
            prob_map[u, v] = variance
            
    # Normalize to form probability distribution
    prob_sum = np.sum(prob_map)
    if prob_sum > 0:
        prob_dist = prob_map / prob_sum
    else:
        prob_dist = np.ones((H, W)) / (H * W)
        
    # Initialize Quadtree
    # Root covers the whole image
    root = QuadtreeNode(0, 0, H, W)
    leaves = [root]
    
    rays = []
    targets = []
    
    # Step 3, 4, 5: Sampling, Rendering, and Subdivision
    # This process is typically iterative over training steps.
    # Here we show one pass of sampling based on current quadtree state.
    
    current_leaves = []
    
    for node in leaves:
        if node.is_marked:
            # Sample n0 rays in this marked node
            # Randomly select n0 pixels within the node bounds
            num_rays = n0
            y_coords = np.random.randint(node.y, node.y + node.height, size=num_rays)
            x_coords = np.random.randint(node.x, node.x + node.width, size=num_rays)
            
            for y, x in zip(y_coords, x_coords):
                ray_origin, ray_direction = create_ray(x, y, camera_params)
                rays.append(ray_origin)
                rays.append(ray_direction)
                targets.append(ground_truth[y, x])
        else:
            # Unmarked node: Sample rays based on probability distribution within the node
            # The number of rays is typically equal to the number of pixels in the node block
            # to maintain resolution, but weighted by probability.
            
            # Extract probability sub-map for this node
            y_start, y_end = node.y, node.y + node.height
            x_start, x_end = node.x, node.x + node.width
            
            node_prob = prob_dist[y_start:y_end, x_start:x_end]
            node_prob_sum = np.sum(node_prob)
            
            if node_prob_sum > 0:
                node_prob_normalized = node_prob / node_prob_sum
            else:
                node_prob_normalized = np.ones_like(node_prob) / (node.height * node.width)
                
            # Flatten probabilities for sampling
            flat_probs = node_prob_normalized.flatten()
            
            # Number of rays to sample = number of pixels in the block
            num_pixels = node.height * node.width
            num_rays = num_pixels
            
            # Sample indices based on probability
            # Add random sampling ratio as per source [S1]
            # The source says "adding a random sampling ratio... to ensure trivial areas... are also learned"
            # This implies mixing uniform random with probability-based.
            # Let's assume a mix ratio or just use the probability distribution which inherently
            # handles complexity, but the source explicitly mentions random sampling ratio.
            # For implementation simplicity based on "probability distribution sampling",
            # we sample indices proportional to flat_probs.
            
            indices = np.random.choice(num_pixels, size=num_rays, p=flat_probs)
            
            for idx in indices:
                local_y = idx // node.width
                local_x = idx % node.width
                global_y = y_start + local_y
                global_x = x_start + local_x
                
                ray_origin, ray_direction = create_ray(global_x, global_y, camera_params)
                rays.append(ray_origin)
                rays.append(ray_direction)
                targets.append(ground_truth[global_y, global_x])
                
            # Store node for subdivision check after rendering
            current_leaves.append(node)
            
    # Step 4 & 5: Error Calculation and Subdivision (Post-rendering)
    # In a real training loop, errors are computed during the forward pass.
    # Here we simulate the logic.
    
    new_leaves = []
    for node in current_leaves:
        # Calculate error e_F for this node
        # This requires accessing the rendered colors for rays in this node
        # Assuming we have a mapping from node to rendered errors
        e_F = calculate_node_error(node, rendered_colors, ground_truth)
        
        if e_F < threshold_alpha:
            node.is_marked = True
            # Do not subdivide
            new_leaves.append(node)
        else:
            # Subdivide into 4 children
            mid_y = node.y + node.height // 2
            mid_x = node.x + node.width // 2
            
            # Top-Left
            child1 = QuadtreeNode(node.x, node.y, mid_x - node.x, mid_y - node.y)
            # Top-Right
            child2 = QuadtreeNode(mid_x, node.y, node.x + node.width - mid_x, mid_y - node.y)
            # Bottom-Left
            child3 = QuadtreeNode(node.x, mid_y, mid_x - node.x, node.y + node.height - mid_y)
            # Bottom-Right
            child4 = QuadtreeNode(mid_x, mid_y, node.x + node.width - mid_x, node.y + node.height - mid_y)
            
            new_leaves.extend([child1, child2, child3, child4])
            
    return rays, targets, new_leaves

class QuadtreeNode:
    def __init__(self, x, y, width, height):
        self.x = x
        self.y = y
        self.width = width
        self.height = height
        self.is_leaf = True
        self.is_marked = False
        self.children = []
```

### Parameters
*   **$\alpha$ (threshold):** The rendering error threshold for marking a node as converged. Not given in the sources.
*   **$n_0$:** Constant number of rays sampled on marked leaf nodes. Typically set to $n_0 = 10$ [S1].
*   **Patch Size:** 3x3 for probability calculation. Alternatives (5x5, 7x7, 9x9) were tested and found insensitive [S1].

### Edge Cases and Pitfalls
*   **Smooth Surfaces:** Context-based probability might ignore trivial areas. The method adds a random sampling ratio to ensure these areas are learned [S1].
*   **Boundary Conditions:** When calculating the 3x3 patch variance, boundary pixels require padding or clamping to avoid index errors [G].
*   **Threshold Selection:** The choice of $\alpha$ affects the trade-off between speed and accuracy. The sources mention an ablation study for this but do not provide the optimal value [S1].

## Operating characteristics
*   **Time Complexity:** Training time is reduced by 15–40% compared to baselines. For example, on the LLFF dataset, Plenoxels training time dropped from 66.6 min to 40.4 min (39% reduction) [S1]. On Tanks and Temples, Instant-NGP dropped from 354s to 285s (19% reduction) [S1]. On DTU, NeuS dropped from 9.2h to 7.3h (21% reduction) [S1].
*   **Memory Complexity:** The quadtree structure adds minimal overhead. The number of rays decreases as training progresses because more nodes become marked and sample only $n_0$ rays instead of full resolution [S1].
*   **Strengths:** Reduces training time significantly while maintaining or improving PSNR/SSIM/LPIPS scores. It is a general framework applicable to NeRF, NeRF++, Plenoxels, Instant-NGP, Mip-NeRF 360, and NeuS [S1].
*   **Limitations:** Initial training time may increase slightly due to the overhead of sampling and subdividing [S1]. The method relies on the quality of the prior probability distribution; if the initial probability map is poor, early sampling may be inefficient.

## Background
The method was proposed in "Fast Learning Radiance Fields by Shooting Much Fewer Rays" [S1]. It addresses the high computational cost of training radiance fields by reducing the number of unnecessary network evaluations in empty or converged regions. It differs from iMap by using error to decide subdivision into 4 children rather than just point count per block, and by allowing the number of leaf nodes to change during training [S1].

## Sources
[S1] Fast Learning Radiance Fields by Shooting Much Fewer Rays, https://doi.org/10.1109/tip.2023.3267049, fetched 2026-10-26
