# Probability-based Ray Sampling with Quadtree Subdivision

Also called: Adaptive Quadtree Subdivision, Fast Learning Radiance Fields by Shooting Much Fewer Rays method, Ours, Quadtree Subdivision

## Overview
This method accelerates radiance field training by dynamically adjusting ray distribution through context-based probability sampling and adaptive quadtree subdivision [S1]. It differs from sibling methods like [Coarse-to-Fine NeRF](3406-coarse-to-fine-nerf.md) by using rendering error on image blocks to trigger structural subdivision into four children, rather than just adjusting point counts within fixed blocks [S1]. Unlike [NeuSample](9472-neusample.md), which focuses on surface normal consistency, this approach explicitly reduces ray counts in converged regions to minimize network evaluations while maintaining or improving reconstruction accuracy [S1].

## How it works
The method operates as a general framework applicable to NeRF, NeRF++, Plenoxels, Instant-NGP, Mip-NeRF 360, and NeuS [S1]. It constructs a spatial hierarchy on each input view to identify regions requiring high sampling density versus those that have converged.

### Inputs and Outputs
*   **Inputs:** Input images $I_i$ with dimensions $H_i \times W_i$; ground truth colors $c_i$; a radiance field model $\Theta$ (e.g., NeRF); a convergence threshold $\alpha$; a constant ray count for marked nodes $n_0$; and a random sampling ratio $\rho$.
*   **Outputs:** A set of sample depths $t_j$ along each ray $r_i$ for volume rendering; an updated radiance field $\Theta$.

### Data Structures
*   **Quadtree:** A hierarchical structure built on each input view. Each node represents a rectangular block of the image.
    *   **Leaf Node:** Represents a block of pixels.
    *   **State:** Each leaf node has a state: `unmarked` (active) or `marked` (converged).
    *   **Subdivision:** An `unmarked` node splits into four equal-sized child nodes. A `marked` node does not subdivide.
*   **Probability Map:** A 2D array of size $H_i \times W_i$ storing the probability density $g'(u, v)$ for each pixel $(u, v)$.

### Parameters
*   $\alpha$: Convergence threshold for average rendering error. Not given in the sources.
*   $n_0$: Constant number of rays sampled on `marked` leaf nodes. Typically $n_0 = 10$ [S1].
*   $\rho$: Random sampling ratio added to the probability distribution to ensure trivial areas are learned. Not given in the sources.
*   Patch Size: Size of the receptive field for variance calculation. Typically $3 \times 3$ [S1].

### Algorithm Steps

1.  **Initialization:**
    For each input view $I_i$ with height $H_i$ and width $W_i$:
    *   Construct a quadtree with a single root node representing the entire image.
    *   Set the initial number of rays $M_i = H_i \times W_i$ [S1].
    *   Calculate the initial probability density $g'(u, v)$ for all pixels $(u, v)$ using the standard deviation of color in a $3 \times 3$ patch [S1].

2.  **Probability Calculation:**
    For each pixel $(u, v)$ in the image:
    *   Define the patch $P_{u,v}$ as the set of pixels $(x, y)$ where $x \in \{u-1, u, u+1\}$ and $y \in \{v-1, v, v+1\}$ [S1].
    *   Calculate the mean color $\bar{c}$ of the patch:
        $$ \bar{c} = \frac{1}{9} \sum_{(x,y) \in P_{u,v}} c(x, y) $$
    *   Calculate the variance (probability density proxy) $g'(u, v)$:
        $$ g'(u, v) = \frac{1}{9} \sum_{(x,y) \in P_{u,v}} [c(x, y) - \bar{c}]^2 $$
    *   [G] Normalize $g'(u, v)$ to form a probability distribution $P(u, v)$ such that $\sum_{u,v} P(u, v) = 1$.
    *   [G] Mix with uniform random sampling: $P_{final}(u, v) = (1 - \rho) P(u, v) + \rho \frac{1}{H_i W_i}$ [S1].

3.  **Ray Sampling and Rendering:**
    For each leaf node $F$ in the quadtree:
    *   If $F$ is `unmarked`:
        *   Sample rays at pixel locations within $F$ according to $P_{final}(u, v)$. The number of rays equals the number of pixels in $F$ [S1].
    *   If $F$ is `marked`:
        *   Sample $n_0$ rays randomly within $F$ [S1].
    *   For each sampled ray $r_i$:
        *   Compute the rendered color $\hat{c}_i = V_\Theta(r_i)$ using volume rendering [S1].
        *   Compute the rendering error for the ray: $e_i = V_\Theta(r_i) - c_i$, where $c_i$ is the ground truth color [S1].

4.  **Error Aggregation and Subdivision:**
    For each leaf node $F$:
    *   Calculate the average rendering error $e_F$ over all rays $r_i$ emitted from $F$:
        $$ e_F = \text{mean}_{r_i \in F} (V_\Theta(r_i) - c_i) $$
    *   If $e_F < \alpha$:
        *   Mark node $F$ as `marked` (converged) [S1].
        *   Do not subdivide $F$ in subsequent iterations [S1].
    *   If $e_F \ge \alpha$:
        *   Subdivide $F$ into four smaller leaf nodes [S1].
        *   Reset the state of the new children to `unmarked` [S1].

5.  **Iteration:**
    *   Repeat steps 2-4 for each training iteration or epoch.
    *   The total number of rays $M_{il}$ after $l$ subdivisions is calculated as:
        $$ M_{il} = Q_{l1} \times \frac{H_i \times W_i}{2^l} + Q_{l2} \times n_0 $$
        where $Q_{l1}$ is the number of `unmarked` leaf nodes and $Q_{l2}$ is the number of `marked` leaf nodes [S1].

### Pseudocode

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def initialize_quadtree(image_height, image_width):
    """
    Creates a quadtree with a single root node covering the entire image.
    Returns a list of leaf nodes.
    """
    root = Node(0, 0, image_height, image_width, state='unmarked')
    return [root]

def calculate_probability_map(image):
    """
    Calculates probability density based on color variance in 3x3 patches.
    """
    h, w, _ = image.shape
    prob_map = np.zeros((h, w))
    
    for u in range(h):
        for v in range(w):
            # Define 3x3 patch boundaries
            x_start = max(0, u - 1)
            x_end = min(h, u + 2)
            y_start = max(0, v - 1)
            y_end = min(w, v + 2)
            
            patch = image[x_start:x_end, y_start:y_end]
            mean_color = np.mean(patch)
            variance = np.mean((patch - mean_color) ** 2)
            prob_map[u, v] = variance
            
    # Normalize to probability distribution
    prob_map = prob_map / np.sum(prob_map)
    return prob_map

def sample_rays_from_node(node, prob_map, n0, random_ratio):
    """
    Samples rays from a leaf node based on its state.
    """
    rays = []
    h, w = node.height, node.width
    y_start, x_start = node.y, node.x
    
    if node.state == 'marked':
        # Sample n0 random rays within the node
        for _ in range(n0):
            y = random.randint(y_start, y_start + h - 1)
            x = random.randint(x_start, x_start + w - 1)
            rays.append((y, x))
    else:
        # Sample rays according to probability distribution
        # Extract sub-probability map for the node
        sub_prob = prob_map[y_start:y_start+h, x_start:x_start+w]
        sub_prob = sub_prob / np.sum(sub_prob) # Re-normalize locally
        
        # Flatten and sample
        pixels = np.array([[y_start + i, x_start + j] for i in range(h) for j in range(w)])
        # Add random sampling ratio to ensure trivial areas are learned
        # [G] Implementation detail: Mix uniform and probability-based sampling
        num_samples = h * w
        indices = np.random.choice(num_samples, size=num_samples, p=sub_prob.flatten())
        
        for idx in indices:
            y, x = pixels[idx]
            rays.append((y, x))
            
    return rays

def update_quadtree(leaf_nodes, rendered_colors, ground_truth_colors, threshold_alpha):
    """
    Updates the quadtree structure based on rendering error.
    """
    new_leaf_nodes = []
    
    for node in leaf_nodes:
        # Calculate average error for this node
        # [G] Assume we have a list of errors for rays in this node
        errors = [rendered_colors[i] - ground_truth_colors[i] for i in node.ray_indices]
        avg_error = np.mean(errors)
        
        if avg_error < threshold_alpha:
            node.state = 'marked'
            new_leaf_nodes.append(node)
        else:
            # Subdivide into 4 children
            h, w = node.height, node.width
            new_h, new_w = h // 2, w // 2
            y, x = node.y, node.x
            
            children = [
                Node(y, x, new_h, new_w, state='unmarked'),
                Node(y, x + new_w, new_h, new_w, state='unmarked'),
                Node(y + new_h, x, new_h, new_w, state='unmarked'),
                Node(y + new_h, x + new_w, new_h, new_w, state='unmarked')
            ]
            new_leaf_nodes.extend(children)
            
    return new_leaf_nodes

def train_step(image, prob_map, leaf_nodes, model, threshold_alpha, n0, random_ratio):
    """
    One step of the training loop with adaptive sampling.
    """
    all_rays = []
    ray_node_map = {} # Map ray index to node for error calculation
    
    for i, node in enumerate(leaf_nodes):
        rays = sample_rays_from_node(node, prob_map, n0, random_ratio)
        for j, (y, x) in enumerate(rays):
            ray_index = len(all_rays)
            all_rays.append((y, x))
            ray_node_map[ray_index] = i
            
    # Render rays
    rendered_colors = model.render(all_rays)
    ground_truth_colors = get_ground_truth_colors(all_rays, image)
    
    # Update model weights
    loss = compute_loss(rendered_colors, ground_truth_colors)
    model.update(loss)
    
    # Update quadtree structure
    leaf_nodes = update_quadtree(leaf_nodes, rendered_colors, ground_truth_colors, threshold_alpha)
    
    return leaf_nodes
```

### Edge Cases and Pitfalls
*   **Threshold Selection:** The choice of threshold $\alpha$ is critical. If too high, nodes remain unmarked and subdivision continues unnecessarily, increasing computation. If too low, nodes are marked prematurely, leading to under-sampling in complex regions [S1].
*   **Smooth Surfaces:** Context-based probability distributions may ignore trivial areas with small color changes. Adding a random sampling ratio $\rho$ is necessary to ensure these areas are learned [S1].
*   **Patch Size Insensitivity:** The method is robust to patch size variations ($3 \times 3$ to $9 \times 9$) and metric choices (variance, entropy, variation), but $3 \times 3$ variance is the standard [S1].

## Subtopics

### Adaptive Quadtree Subdivision
The core mechanism of splitting leaf nodes based on rendering error. It differs from fixed-block methods by allowing the number of blocks to change dynamically [S1].

### Context-Based Probability Sampling
The use of local color variance to bias ray sampling towards complex regions. It differs from uniform sampling by focusing computational effort on high-detail areas [S1].

## Operating characteristics
*   **Time Complexity:** Training time is reduced by 15-40% compared to baseline methods [S1].
    *   On LLFF dataset (Plenoxels): 66.6 min to 40.4 min (39% reduction) [S1].
    *   On Tanks and Temples (Instant-NGP): 354s to 285s (19% reduction) [S1].
    *   On DTU (NeuS): 9.2h to 7.3h (21% reduction) [S1].
*   **Memory Complexity:** The quadtree structure adds minimal overhead. The number of rays $M_{il}$ decreases as training progresses because marked nodes sample fewer rays ($n_0$) compared to unmarked nodes [S1].
*   **Performance:** Achieves comparable or better PSNR, SSIM, and LPIPS scores than baselines [S1].
    *   LLFF (Plenoxels): PSNR 24.60 to 25.12 [S1].
    *   Tanks and Temples (Instant-NGP): PSNR 27.91 to 28.25 [S1].
    *   DTU (NeuS): Chamfer Distance 0.87 to 0.73 [S1].
*   **Strengths:** General framework applicable to various radiance field methods; significant speedup in later training stages; maintains high accuracy [S1].
*   **Limitations:** Initial training phase may have slightly increased time due to sampling and subdividing overhead [S1].

## Background
The method was proposed in "Fast Learning Radiance Fields by Shooting Much Fewer Rays" [S1]. It addresses the high computational cost of training radiance fields by reducing the number of rays needed for convergence. It builds on the principles of adaptive ray sampling [8870-adaptive-ray-sampling.md] and differs from prior work like iMap by using error-driven subdivision rather than fixed block allocation [S1].

## Sources
[S1] Fast Learning Radiance Fields by Shooting Much Fewer Rays, https://doi.org/10.1109/tip.2023.3267049, fetched 2026-10-26
