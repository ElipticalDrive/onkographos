# Probability-based Ray Sampling with Quadtree Subdivision

Also called: Adaptive Quadtree Subdivision, Fast Learning Radiance Fields by Shooting Much Fewer Rays method, Ours, Quadtree Subdivision

## Overview
This method accelerates radiance field training by dynamically adjusting ray distribution using a context-based probability density and an adaptive quadtree subdivision strategy [S1]. It reduces training time by identifying converged regions (low error) to sample fewer rays, while refining complex regions (high error) through recursive subdivision [S1]. It differs from sibling methods like [Mip-NeRF 360 Sampling and Training](3393-mip-nerf-360-sampling-and-training.md) and [Coarse-to-Fine NeRF](3406-coarse-to-fine-nerf.md) by using a hierarchical spatial partition (quadtree) driven by per-node rendering error rather than uniform blocks or fixed coarse/fine networks [S1].

## How it works
**Inputs and Outputs:**
*   **Inputs:** Input views (images) with dimensions $H_i \times W_i$; a radiance field model $\Theta$ (e.g., NeRF, Plenoxels, Instant-NGP, NeuS); a convergence threshold $\alpha$; a constant ray count for marked nodes $n_0$; and a random sampling ratio $\rho$ [S1].
*   **Outputs:** A set of rays to evaluate for the current training step, defined by the current quadtree structure [S1].

**Data Structures:**
*   **Quadtree:** A hierarchical structure built on each input view. Each node represents a rectangular block of the image.
    *   **Leaf Node:** Represents a block of pixels. Contains a state flag `marked` (boolean).
    *   **Internal Node:** Represents a parent block, subdivided into four children.
*   **Probability Map:** A 2D array of size $H_i \times W_i$ storing the prior probability density for each pixel, calculated from image context [S1].

**Step-by-Step Procedure:**

1.  **Initialization:**
    *   Construct a quadtree for each input view $I_i$. Initially, the tree consists of a single root node covering the entire image ($H_i \times W_i$).
    *   Calculate the prior probability distribution $g'(u, v)$ for every pixel $(u, v)$ in the image based on color variance [S1].
    *   Set the initial number of rays $M_i = H_i \times W_i$ (one ray per pixel) [S1].

2.  **Probability Calculation (Context-Based Sampling):**
    *   For each pixel $(u, v)$, calculate the variance of color values in a $3 \times 3$ patch centered at $(u, v)$ [S1].
    *   The variance is defined as:
        $$ \text{variance}(c(u, v)) = \frac{1}{9} \sum_{x \in \{u-1, u, u+1\}} \sum_{y \in \{v-1, v, v+1\}} [c(x, y) - \bar{c}]^2 $$
        where $\bar{c}$ is the mean color in the patch [S1].
    *   Normalize these variances across the image to form a probability distribution.
    *   **Random Sampling Component:** To prevent ignoring trivial but important areas (e.g., smooth surfaces), mix in a random sampling ratio $\rho$. The final probability of selecting a pixel is a combination of the context-based variance and uniform random selection [S1].

3.  **Ray Sampling and Rendering:**
    *   Traverse all leaf nodes in the quadtree.
    *   For each leaf node $F$:
        *   If $F$ is **marked** (converged): Sample only $n_0$ rays uniformly or randomly within the block $F$ [S1].
        *   If $F$ is **unmarked**: Sample rays within the block $F$ according to the prior probability distribution $g'(u, v)$ calculated in Step 2. The number of rays sampled in an unmarked node is typically equal to the number of pixels in that node (or proportional to its area) [S1].
    *   Shoot the selected rays through the radiance field $\Theta$ to render colors $\hat{c}$ [S1].

4.  **Error Calculation:**
    *   For each leaf node $F$, calculate the average rendering error $e_F$ over all rays $r_i$ emitted from pixels in $F$:
        $$ e_F = \text{mean}_{r_i \in F} (V_\Theta(r_i) - c_i) $$
        where $V_\Theta(r_i)$ is the rendered color and $c_i$ is the ground truth color [S1].

5.  **Quadtree Subdivision:**
    *   Compare $e_F$ with the threshold $\alpha$ [S1].
    *   If $e_F < \alpha$: Mark leaf node $F$ as **converged** (`marked = true`). Do not subdivide. Future iterations will sample only $n_0$ rays from this node [S1].
    *   If $e_F \ge \alpha$: Subdivide node $F$ into four smaller child leaf nodes. These new children are initially unmarked. This allows for higher resolution sampling in complex regions [S1].

6.  **Iteration:**
    *   Repeat Steps 3-5 for subsequent training epochs. The total number of rays $M_i$ decreases over time as more nodes become marked [S1].
    *   The total number of rays after $l$ subdivisions is given by:
        $$ M_{il} = Q^l_1 \times \frac{H_i \times W_i}{2^l} + Q^l_2 \times n_0 $$
        where $Q^l_1$ is the number of unmarked leaf nodes and $Q^l_2$ is the number of marked leaf nodes at subdivision level $l$ [S1].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def initialize_quadtree(image_height, image_width):
    """
    Creates a single root node covering the entire image.
    Returns a list of leaf nodes (initially just the root).
    """
    root = Node(x=0, y=0, w=image_height, h=image_width, marked=False)
    return [root]

def calculate_probability_map(image):
    """
    Calculates probability density for each pixel using 3x3 color variance.
    Returns a normalized 2D array of probabilities.
    """
    h, w, _ = image.shape
    prob_map = np.zeros((h, w))
    
    # Calculate variance for each pixel in a 3x3 patch
    for u in range(h):
        for v in range(w):
            # Define patch boundaries, handling edges by clamping or padding
            u_start = max(0, u - 1)
            u_end = min(h, u + 2)
            v_start = max(0, v - 1)
            v_end = min(w, v + 2)
            
            patch = image[u_start:u_end, v_start:v_end]
            mean_c = np.mean(patch)
            variance = np.mean((patch - mean_c) ** 2)
            prob_map[u, v] = variance
            
    # Normalize to sum to 1
    prob_map /= np.sum(prob_map)
    return prob_map

def sample_rays_from_leaf(leaf_node, prob_map, n0, random_ratio):
    """
    Samples rays from a leaf node.
    If marked, samples n0 rays randomly.
    If unmarked, samples rays based on prob_map within the node's bounds.
    """
    rays = []
    h_start, w_start = leaf_node.y, leaf_node.x
    h_end, w_end = leaf_node.y + leaf_node.h, leaf_node.x + leaf_node.w
    
    if leaf_node.marked:
        # Sample n0 rays randomly within the node
        for _ in range(n0):
            u = random.randint(h_start, h_end - 1)
            v = random.randint(w_start, w_end - 1)
            rays.append((u, v))
    else:
        # Extract sub-probability map for this node
        sub_prob = prob_map[h_start:h_end, w_start:w_end]
        # Normalize sub-prob
        sub_prob /= np.sum(sub_prob)
        
        # Number of pixels in this node
        num_pixels = leaf_node.h * leaf_node.w
        
        # Sample based on probability distribution
        # Flatten indices for sampling
        flat_indices = np.arange(num_pixels)
        flat_probs = sub_prob.flatten()
        
        # Add random sampling component (as per ablation study)
        # Mix uniform random with probability-based
        # Note: Exact mixing formula not specified, assuming weighted combination
        # or separate sampling pools. Here we assume standard inverse transform
        # on the combined distribution.
        
        sampled_indices = np.random.choice(flat_indices, size=num_pixels, p=flat_probs)
        for idx in sampled_indices:
            u = h_start + idx // leaf_node.w
            v = w_start + idx % leaf_node.w
            rays.append((u, v))
            
    return rays

def update_quadtree(leaf_nodes, rendered_colors, ground_truth_colors, threshold_alpha):
    """
    Evaluates error for each leaf node and subdivides if necessary.
    Returns updated list of leaf nodes.
    """
    new_leaves = []
    
    for leaf in leaf_nodes:
        # Calculate average error for this leaf
        # Note: rendered_colors and ground_truth_colors must be aligned with rays sampled from this leaf
        # Assuming we have a list of errors for pixels in this leaf
        errors = []
        for (u, v) in leaf.rays_sampled: # rays_sampled is stored from previous step
            err = abs(rendered_colors[u, v] - ground_truth_colors[u, v])
            errors.append(err)
            
        if not errors:
            # Fallback if no rays were sampled (e.g., empty node)
            avg_error = 0
        else:
            avg_error = np.mean(errors)
            
        if avg_error < threshold_alpha:
            leaf.marked = True
            new_leaves.append(leaf)
        else:
            # Subdivide into 4 children
            h, w = leaf.h, leaf.w
            new_h, new_w = h // 2, w // 2
            
            # Create 4 children
            children = [
                Node(leaf.x, leaf.y, new_w, new_h, marked=False),
                Node(leaf.x + new_w, leaf.y, new_w, new_h, marked=False),
                Node(leaf.x, leaf.y + new_h, new_w, new_h, marked=False),
                Node(leaf.x + new_w, leaf.y + new_h, new_w, new_h, marked=False)
            ]
            new_leaves.extend(children)
            
    return new_leaves

def train_step(image, quadtree_leaves, model, prob_map, n0, threshold_alpha, random_ratio):
    """
    One full training step involving sampling, rendering, and updating.
    """
    all_rays = []
    
    # 1. Sample rays from all current leaves
    for leaf in quadtree_leaves:
        rays = sample_rays_from_leaf(leaf, prob_map, n0, random_ratio)
        leaf.rays_sampled = rays # Store for error calculation
        all_rays.extend(rays)
        
    # 2. Render using model
    rendered_colors = render_rays(model, all_rays)
    
    # 3. Get ground truth colors for these rays
    ground_truth_colors = get_ground_truth(image, all_rays)
    
    # 4. Update quadtree structure based on error
    quadtree_leaves = update_quadtree(quadtree_leaves, rendered_colors, ground_truth_colors, threshold_alpha)
    
    return quadtree_leaves
```

**Parameters:**
*   $\alpha$ (threshold): Convergence threshold for marking nodes. Not given in the sources. [S1]
*   $n_0$: Constant number of rays sampled on marked leaf nodes. Typically set to $n_0 = 10$ [S1].
*   $\rho$ (random sampling ratio): Ratio of random sampling added to the probability distribution. Not given in the sources. [S1]
*   Patch Size: $3 \times 3$ is used. Alternative sizes ($5 \times 5, 7 \times 7, 9 \times 9$) and metrics (variation, entropy) were tested but found insensitive [S1].

**Edge Cases and Pitfalls:**
*   **Boundary Handling:** The $3 \times 3$ patch calculation requires handling image boundaries (clamping or padding) [S1].
*   **Initial Overhead:** There is a tiny increase in training time at the beginning due to sampling and subdividing overhead [S1].
*   **Trivial Areas:** Without random sampling, context-based probability may ignore trivial but important areas (smooth surfaces) [S1].

## Operating characteristics
*   **Time Complexity:** Training time is reduced by 15-40% compared to baselines (NeRF, Plenoxels, Instant-NGP, NeuS) [S1]. The reduction increases as training progresses because the number of unmarked nodes ($Q^l_1$) grows slower than the number of marked nodes ($Q^l_2$), leading to fewer total rays [S1].
*   **Memory Complexity:** Not given in the sources.
*   **Measured Performance:**
    *   **LLFF (Forward-Facing):** Training time reduced from 66.6 min (Plenoxels) to 40.4 min (39% reduction). PSNR increased from 24.60 to 25.12 [S1].
    *   **Tanks and Temples:** Instant-NGP training time reduced from 354s to 285s (19% reduction). PSNR increased from 27.91 to 28.25 [S1].
    *   **DTU:** NeuS training time reduced from 9.2h to 7.3h (21% reduction). Chamfer Distance improved from 0.87 to 0.73 [S1].
*   **Strengths:** General framework applicable to NeRF, NeRF++, Plenoxels, Instant-NGP, Mip-NeRF 360, and NeuS. Balances efficiency and effectiveness, improving performance at early stages and speed at later stages [S1].
*   **Limitations:** Initial overhead for subdivision. Performance is sensitive to the choice of random sampling ratio and threshold $\alpha$ (though patch size is insensitive) [S1].

## Background
*   **Origin:** Proposed in "Fast Learning Radiance Fields by Shooting Much Fewer Rays" [S1].
*   **History:** The method is presented as a general framework to accelerate radiance field training. It builds on the concept of using rendering error to guide sampling, similar to iMap, but introduces adaptive quadtree subdivision instead of fixed blocks [S1].
*   **Usage:** Applied to accelerate various radiance field methods including NeRF, NeRF++, Plenoxels, Instant-NGP, Mip-NeRF 360, and NeuS [S1].

## Subtopics

### Adaptive Ray Sampling
The parent method providing the general framework for non-uniform distribution and hierarchical evaluation. [Adaptive Ray Sampling](8870-adaptive-ray-sampling.md)

### Mip-NeRF 360 Sampling and Training
Uses conical frustums and multi-resolution hashing, differing from the quadtree error-driven subdivision. [Mip-NeRF 360 Sampling and Training](3393-mip-nerf-360-sampling-and-training.md)

### Coarse-to-Fine NeRF
Uses a two-stage network (coarse/fine) rather than a spatial quadtree partition. [Coarse-to-Fine NeRF](3406-coarse-to-fine-nerf.md)

### Hard Sample Mining
Focuses on mining hard pixels based on loss, but does not use a hierarchical spatial subdivision structure. [Hard Sample Mining](8673-hard-sample-mining.md)

### TermiNeRF
Uses termination criteria based on depth, differing from the error-based quadtree subdivision. [TermiNeRF](8828-terminerf.md)

### Pixel Region and Depth Guided Ray Sampling
Uses depth and pixel regions for guidance, not quadtree error subdivision. [Pixel Region and Depth Guided Ray Sampling](8866-pixel-region-and-depth-guided-ray-sampling.md)

### NeRF-ID
Uses a proposer network for sampling, differing from the explicit quadtree structure. [NeRF-ID](8867-nerf-id.md)

### Two-Stage Sampling with Occupancy Grid and Nonlinear Function
Uses occupancy grids and nonlinear functions, not quadtree subdivision. [Two-Stage Sampling with Occupancy Grid and Nonlinear Function](9146-two-stage-sampling-with-occupancy-grid-and-nonlinear-functio.md)

### Truncated Depth-Based Sampling
Uses depth truncation, not error-driven subdivision. [Truncated Depth-Based Sampling](10613-truncated-depth-based-sampling.md)

### Superpixel-guide Efficient Sampling (SES)
Uses superpixels for guidance, differing from the quadtree approach. [Superpixel-guide Efficient Sampling (SES)](11252-superpixel-guide-efficient-sampling-ses.md)

### Surface-Based Sampling
Samples near surfaces, not based on image block error. [Surface-Based Sampling](11356-surface-based-sampling.md)

## Sources
[S1] Fast Learning Radiance Fields by Shooting Much Fewer Rays, https://doi.org/10.1109/tip.2023.3267049, fetched 2026-10-26
