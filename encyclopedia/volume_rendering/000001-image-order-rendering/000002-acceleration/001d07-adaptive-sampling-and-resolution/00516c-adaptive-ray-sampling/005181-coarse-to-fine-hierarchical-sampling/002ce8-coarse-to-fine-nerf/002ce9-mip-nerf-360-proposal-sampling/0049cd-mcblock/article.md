# MCBlock

Also called: MCTS-based Dynamic-Resolution Ray Sampling

## Overview
MCBlock is a dynamic-resolution ray-sampling algorithm for Neural Radiance Field (NeRF) training that partitions training images into pixel blocks of varying sizes based on texture intensity, using Monte Carlo Tree Search (MCTS) to optimize the partition during training [S1, unofficial]. It accelerates training by focusing high-frequency sampling on complex regions and low-frequency sampling on simple regions, addressing the inefficiency of uniform pixel-wise sampling and the artifacts of static multi-resolution methods [S1, unofficial]. Unlike its siblings, it combines active sampling with dynamic block partitioning and block-wise novel view synthesis to eliminate high-frequency flaws in simple-texture regions [S1, unofficial].

## How it works
**Inputs and Outputs:**
*   **Inputs:** A set of training images with known camera poses; a cone-tracing NeRF model (e.g., Mipnerf, Zipnerf); hyperparameters $\lambda$ (temperature coefficient), $\epsilon_L$ (pruning threshold), $\epsilon_{init}$ (initialization variance threshold), $\epsilon_C$ (complexity threshold) [S1, unofficial].
*   **Outputs:** A trained NeRF model; accelerated training time; novel view images rendered in a block-wise manner [S1, unofficial].

**Data Structures:**
*   **Monte-Carlo Quadtree:** Each training image has an associated quadtree [S1, unofficial].
    *   **Root Node:** Represents the entire image of size $H \times W$ [S1, unofficial].
    *   **Internal Node:** Represents a rectangular pixel block. Has four children representing evenly divided sub-blocks (e.g., $(H/2, W/2)$, $(H/4, W/4)$, etc.) [S1, unofficial].
    *   **Leaf Node:** Represents a pixel block currently selected for training [S1, unofficial].
    *   **Node Fields:**
        *   $|B_i|$: Size of block $i$ (height $\times$ width) [S1, unofficial].
        *   $L_i$: Loss value of block $i$. For leaf nodes, $L_i = \|C_{B_i} - C^c_{B_i}\|_2$ (MSE between average color $C_{B_i}$ and recently rendered color $C^c_{B_i}$) [S1, unofficial].
        *   $U_i$: UCT value. For leaf nodes, $U_i = L_i \cdot e^{\lambda \cdot O_i} \cdot |B_i|$, where $O_i$ is the time interval since last sampling [S1, unofficial]. For internal nodes, $U_j = \sum_{k \in Ch_j} U_k$ [S1, unofficial].
        *   $O_i$: Expired time (time since last sampling). Reset to 0 upon sampling [S1, unofficial].

**Parameters:**
*   $\lambda$: Temperature coefficient controlling the influence of the expired time term in UCT calculation. Typical value: Not explicitly given, but used in formulas [S1, unofficial].
*   $\epsilon_L$: Threshold for scaled loss during pruning. Typical value: $10^{-2}$ [S1, unofficial].
*   $\epsilon_{init}$: Threshold for texture variance during initialization. Typical value: $10^{-3}$ [S1, unofficial].
*   $\epsilon_C$: Threshold for complexity/expansion. Typical value: $10^{-4}$ [S1, unofficial].

**Algorithm Steps:**

1.  **Initialization:**
    *   For each training image, compute color variance [S1, unofficial].
    *   Initialize the quadtree structure based on texture intensity. Regions with variance below $\epsilon_{init}$ are assigned larger initial blocks; complex regions are assigned smaller blocks [S1, unofficial].
    *   All leaf nodes form the initial block partition [S1, unofficial].

2.  **Block Selection (Tree Walk):**
    *   Start at the root node [S1, unofficial].
    *   At each internal node $j$, select a child node $k$ with probability proportional to its UCT value relative to the sum of UCT values of all children: $P(k) = \frac{U_k}{\sum_{m \in Ch_j} U_m}$ [S1, unofficial].
    *   Continue until a leaf node $j$ is selected [S1, unofficial].

3.  **Block Rendering and Loss Calculation:**
    *   Render the selected leaf block $B_j$ using the cone-tracing NeRF model [S1, unofficial].
    *   Calculate the rendering loss $L_j = \|C_{B_j} - C^c_{B_j}\|_2$ [S1, unofficial].
    *   Perform backpropagation through the NeRF model to update weights [S1, unofficial].

4.  **Backpropagation and Pruning:**
    *   Multiply all UCT values in the tree by $e^\lambda$ to simulate the increment of expired time $O$ [S1, unofficial].
    *   For each leaf node $j$ in the current batch:
        *   Set $O_j = 0$.
        *   Set $U_j = L_j \cdot |B_j|$ [S1, unofficial].
    *   Walk up from each leaf node to the root:
        *   For each internal node $j$ encountered:
            *   Update $U_j = \sum_{k \in Ch_j} U_k$.
            *   Update $L_j = \frac{1}{|Ch_j|} \sum_{k \in Ch_j} L_k$ [S1, unofficial].
    *   If batch size is $S_{batch}$ and image size is $S_{image}$, this process contains complex operations for $\log_4(S_{image}) \cdot S_{batch}$ nodes and simple multiplications for all nodes, ensuring high efficiency [S1, unofficial].
    *   **Pruning:** Prune children of internal nodes if their scaled loss falls below $\epsilon_L$ to promote training efficiency [S1, unofficial]. *Note: The exact pruning condition formula is not explicitly provided in the text, but it relies on $\epsilon_L$ and loss values.*

5.  **Expansion:**
    *   If the selected leaf node $j$ has high loss (indicating complex texture or high-frequency flaws), split its block into four equal sub-blocks (children) [S1, unofficial].
    *   Assign the UCT value (scaled by block size) and loss value of node $j$ to the new child nodes to bypass expensive initial loss calculations [S1, unofficial].
    *   Randomly add one of the newly expanded nodes to the training batch for the current iteration [S1, unofficial].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import math
import random

class Node:
    def __init__(self, height, width, parent=None):
        self.height = height
        self.width = width
        self.parent = parent
        self.children = []
        self.UCT = 0.0
        self.loss = 0.0
        self.expired_time = 0.0

def compute_variance(image):
    """Compute color variance of an image array."""
    return float(image.var())

def initialize_tree(image_variance, height, width, epsilon_init):
    """
    Initialize quadtree based on texture intensity.
    The source describes texture-based initialization to boost speed.
    Exact recursive splitting logic is not provided in code form.
    """
    root = Node(height, width)
    # Placeholder for recursive variance-based splitting logic
    return root

def select_leaf(root):
    """Select a leaf node using UCT probabilities."""
    node = root
    while node.children:
        sum_uct = sum(child.UCT for child in node.children)
        if sum_uct == 0:
            node = node.children[0]
        else:
            r = random.uniform(0, sum_uct)
            cumulative = 0.0
            for child in node.children:
                cumulative += child.UCT
                if cumulative >= r:
                    node = child
                    break
    return node

def render_block(block_node, nerf_model):
    """
    Render a block using cone-tracing NeRF.
    Returns average color and rendered color tensor.
    """
    avg_color, rendered_colors = nerf_model.forward(block_node)
    return avg_color, rendered_colors

def calculate_loss(avg_color, rendered_colors):
    """Calculate MSE loss between average color and rendered colors."""
    return float(((avg_color - rendered_colors) ** 2).mean())

def backpropagate_and_prune(root, batch_leaves, lambda_param, epsilon_L):
    """
    Update UCT and loss values, and prune nodes.
    """
    # Multiply all UCT values by e^lambda
    def update_expired_time(node):
        node.UCT *= math.exp(lambda_param)
        for child in node.children:
            update_expired_time(child)
    
    update_expired_time(root)
    
    # Reset expired time and update UCT for batch leaves
    for leaf in batch_leaves:
        leaf.expired_time = 0
        leaf.UCT = leaf.loss * (leaf.height * leaf.width)
        
    # Walk up and update internal nodes
    for leaf in batch_leaves:
        current = leaf
        while current.parent:
            parent = current.parent
            parent.UCT = sum(child.UCT for child in parent.children)
            parent.loss = sum(child.loss for child in parent.children) / len(parent.children)
            current = parent
            
    # Pruning logic (simplified based on description)
    # If a node's loss is below threshold, it might be pruned (merged with siblings)
    # The exact pruning condition is not fully specified in the source text.
    pass

def expand_node(node, batch):
    """
    Split a leaf node into four children if loss is high.
    """
    h, w = node.height, node.width
    new_h, new_w = h // 2, w // 2
    
    # Create four children
    children = []
    for _ in range(4):
        child = Node(new_h, new_w, parent=node)
        # Assign UCT and loss from parent. Source says "scaled by block size".
        child.UCT = node.UCT / 4
        child.loss = node.loss
        children.append(child)
        
    node.children = children
    node.UCT = 0
    node.loss = 0
    
    # Randomly add one expanded node to batch
    batch.append(random.choice(children))

def train_mcblock(nerf_model, images, poses, lambda_param, epsilon_L, epsilon_init, epsilon_C, batch_size, num_iterations):
    root = initialize_tree(compute_variance(images[0]), images[0].shape[0], images[0].shape[1], epsilon_init)
    
    for iteration in range(num_iterations):
        batch_leaves = []
        
        # Selection and Expansion
        for _ in range(batch_size):
            leaf = select_leaf(root)
            # Render and calculate loss
            avg_color, rendered_colors = render_block(leaf, nerf_model)
            loss = calculate_loss(avg_color, rendered_colors)
            leaf.loss = loss
            
            # Backpropagate through NeRF
            nerf_model.backward(loss)
            
            batch_leaves.append(leaf)
            
            # Expansion if loss is high
            if loss > epsilon_C:
                expand_node(leaf, batch_leaves)
                
        # Backpropagate UCT and Loss in the tree
        backpropagate_and_prune(root, batch_leaves, lambda_param, epsilon_L)
```

**Edge Cases and Pitfalls:**
*   **High-frequency flaws in simple-texture regions:** Without block splitting, simple regions may lack supervision for high-frequency parameters, leading to artifacts like floaters [S1, unofficial].
*   **Novel view synthesis artifacts:** Rendering novel views pixel-wise can expose faulty parameters [S1, unofficial]. MCBlock uses block-wise rendering, warping the training view block partition to the novel view, to avoid rendering high-frequency flaws [S1, unofficial].
*   **Testing view performance:** MCBlock performance on testing views may be slightly worse than baselines because the estimated block partition for testing views is not precise [S1, unofficial].
*   **Precision and Boundaries:** The algorithm relies on accurate cone-tracing and IPE [S1, unofficial].

## Operating characteristics
*   **Time Complexity:** The backpropagation and UCT update process involves complex operations for $\log_4(S_{image}) \cdot S_{batch}$ nodes and simple multiplications for all nodes, ensuring high efficiency [S1, unofficial].
*   **Memory Complexity:** Requires storing the quadtree structure for each training image, with node counts proportional to the image resolution and the depth of the tree [S1, unofficial].
*   **Measured Performance:**
    *   Achieves up to 2.33x training acceleration on the Blender dataset (LPIPS = 0.030) compared to random sampling [S1, unofficial].
    *   Achieves up to 1.66x acceleration on outdoor Mipnerf360 scenes (LPIPS = 0.145) and 1.6x on indoor Mipnerf360 scenes (LPIPS = 0.035) compared to random sampling [S1, unofficial].
    *   Reduces training time to 47.5% of random ray selection on Mipnerf/lego and 51.3% on Zipnerf/bicycle [S1, unofficial].
    *   Overhead is 5.9% of total iteration time for Zipnerf and 4.1% for Mipnerf on RTX 4090 [S1, unofficial].
*   **Strengths:** Accelerates training significantly, especially in early stages; eliminates high-frequency artifacts in simple-texture regions; works with any cone-tracing NeRF model [S1, unofficial].
*   **Limitations:** Slightly worse performance on testing views due to imprecise block partition estimation; advantage over other methods diminishes in later training stages as loss values become outdated [S1, unofficial].
*   **When to use:** When training speed is a priority and the scene contains regions with varying texture complexity (e.g., outdoor scenes with sky/background) [S1, unofficial]. Not recommended when testing view quality is the sole metric and computational resources are not a constraint [S1, unofficial].

## Background
MCBlock was proposed by Yunpeng Tan, Junlin Hao, Jiangkai Wu, Liming Liu, Qingyang Li, and Xinggong Zhang [S1, unofficial]. It is implemented in Nerfstudio and tested on Mipnerf360 and Blender datasets [S1, unofficial]. The method builds on the concept of cone-tracing NeRFs (like Mipnerf) which use Integrated Positional Encoding (IPE) to handle low-resolution rendering, and active ray sampling methods like ActRay, which sample pixels based on fitting difficulty [S1, unofficial]. MCBlock extends this by using MCTS to dynamically partition images into blocks of different sizes, addressing the limitations of uniform block sizes in PyNeRF and pixel-wise sampling in ActRay [S1, unofficial].

## Sources
[S1] MCBlock: Boosting Neural Radiance Field Training Speed by MCTS-based Dynamic-Resolution Ray Sampling, https://doi.org/10.48550/arXiv.2504.09878, fetched 2026-10-27 (unofficial)
