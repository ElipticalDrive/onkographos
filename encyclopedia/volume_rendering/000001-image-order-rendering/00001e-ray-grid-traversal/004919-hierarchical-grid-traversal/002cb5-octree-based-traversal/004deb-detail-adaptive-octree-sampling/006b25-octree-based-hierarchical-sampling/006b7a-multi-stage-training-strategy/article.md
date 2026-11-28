# Multi-stage Training Strategy

Also called: Iterative Octree Reconstruction Training, Octree Reconstruction, Stage-based Octree Refinement

## Overview
The Multi-stage Training Strategy is a training procedure for octree-based volumetric super-resolution that divides training into sequential stages, reconstructing the octree at the end of each stage to address the failure of importance sampling as model weights update [S1]. It sets itself apart from single-stage hierarchical sampling by dynamically adapting the sampling distribution to the current model's error landscape, ensuring that subsequent stages continue to benefit from importance sampling rather than converging prematurely on an outdated distribution [S1].

## How it works
**Inputs and Outputs:**
*   **Inputs:** Ground truth volumetric data $D_{gt}$, initial octree (built from variance), current best model $N^k_{best}$, minimum epoch count $q$, loss threshold $Q$.
*   **Outputs:** Final trained model, final octree structure.

**Data Structures:**
*   **Octree:** Hierarchical tree where each node represents a cuboid subblock. Leaf nodes are indivisible. Nodes store regression loss aggregates for subdivision decisions.
*   **Model:** Neural network $N$ (encoder-decoder) mapping low-resolution inputs to high-resolution outputs.

**Parameters:**
*   $q$: Minimum number of epochs per stage. Typical value: $q \ge 10$ [S1].
*   $Q$: Loss threshold for termination. Not given in the sources.
*   $a$: Downsampling scale factor for inference. Not given in the sources.
*   $s$: Minimum dimension size for subdivision. Not given in the sources.

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def multi_stage_training(D_gt, initial_octree, N_best, q, Q, a):
    """
    D_gt: Ground truth volumetric data
    initial_octree: Octree initialized via variance
    N_best: Current best model (initially trained model or None)
    q: Minimum epochs per stage
    Q: Loss threshold for termination
    a: Downsampling scale factor
    """
    k = 0  # Stage counter
    octree = initial_octree
    
    while True:
        # 1. Train for one stage
        # Start from epoch 1 (or continue if resuming, but logic implies fresh stage start)
        # The source implies training runs until bottleneck detection
        epoch = 0
        loss_history = []
        
        while True:
            epoch += 1
            
            # Standard training step for one epoch
            # - Randomly crop cubic data blocks from D_gt
            # - Downsample to get D_a_lr
            # - Encode to latent context grid
            # - Perform hierarchical sampling using current octree
            # - Decode and compute loss (Regression + PDE)
            # - Update model weights
            current_loss = train_one_epoch(D_gt, octree, N_best, a)
            loss_history.append(current_loss)
            
            # Check termination conditions only after minimum epochs
            if epoch < q:
                continue
            
            # 2. Detect Bottleneck
            # Use loss values of the last 10 epochs
            recent_losses = loss_history[-10:]
            mean_loss, slope = linear_regression(recent_losses)
            
            # Condition A: Terminate training entirely
            if mean_loss <= Q:
                return N_best, octree
            
            # Condition B: Start next stage (reconstruct octree)
            # Mean is high AND slope is non-negative (loss not decreasing)
            if mean_loss > Q and slope >= 0:
                break
        
        # 3. Reconstruct Octree
        k += 1
        octree = reconstruct_octree(D_gt, N_best, a, octree)
        
        # Note: N_best is updated during training, so it reflects the best model 
        # from the previous stage. The source uses N^k_best for inference.

def reconstruct_octree(D_gt, N_best, a, current_octree):
    """
    Rebuilds the octree based on regression loss distribution from the current best model.
    """
    # 1. Downsample ground truth for faster inference
    D_a_gt = downsample(D_gt, a)
    
    # 2. Perform inference to get high-resolution output
    # D^k_hr = N^k_best(D_a_lr)
    D_a_lr = D_a_gt  # Assuming D_a_lr is the downsampled version used as input
    D_hr = N_best(D_a_lr)
    
    # 3. Compute regression loss distribution
    # L^k_j = (1/c) * sum(alpha_i * ||x_ij - x_hat^k_ij||_1)
    # L^k_tot = (1/n) * sum(L^k_j)
    n = number_of_grid_points(D_a_gt)
    c = number_of_physical_variables()
    alpha = get_variable_weights() # Not given in sources, assumed known
    
    total_loss = 0.0
    grid_losses = []
    
    for j in range(n):
        # Get estimated value from D_hr
        x_hat = get_value_at_grid_point(D_hr, j)
        # Get ground truth value
        x_true = get_value_at_grid_point(D_a_gt, j)
        
        # Compute L1 loss for each variable
        var_loss = 0.0
        for i in range(c):
            var_loss += alpha[i] * abs(x_true[i] - x_hat[i])
        
        L_j = var_loss / c
        grid_losses.append(L_j)
        total_loss += L_j
        
    L_tot = total_loss / n
    
    # 4. Recursively reconstruct octree from root
    # Subdivide if L_sub,k_tot >= L_tot AND at least one dimension > s
    new_octree = Octree()
    new_octree.root = Subdivide(D_a_gt, D_hr, grid_losses, L_tot, current_octree.root)
    
    return new_octree

def Subdivide(D_gt, D_hr, grid_losses, L_tot, node):
    """
    Recursively subdivides octree nodes based on loss.
    """
    # Calculate subblock regression loss L_sub,k_tot
    # Similar to Equation 6, summing L_j for points in this subblock
    subblock_points = get_points_in_subblock(node)
    L_sub_tot = sum(grid_losses[j] for j in subblock_points) / len(subblock_points)
    
    # Check subdivision conditions
    # 1. Loss condition: L_sub,k_tot >= L_tot
    # 2. Size condition: at least one dimension > s
    if L_sub_tot >= L_tot and has_dimension_greater_than(node, s):
        # Subdivide
        children = create_children(node)
        for child in children:
            Subdivide(D_gt, D_hr, grid_losses, L_tot, child)
        return children
    else:
        return node

def linear_regression(losses):
    """
    Performs linear regression on the last 10 loss values.
    Returns (mean_value, slope).
    """
    # Simple linear regression implementation
    n = len(losses)
    x = list(range(n))
    y = losses
    
    mean_x = sum(x) / n
    mean_y = sum(y) / n
    
    numerator = sum((x[i] - mean_x) * (y[i] - mean_y) for i in range(n))
    denominator = sum((x[i] - mean_x) ** 2 for i in range(n))
    
    if denominator == 0:
        slope = 0
    else:
        slope = numerator / denominator
        
    # The "mean value" in the source likely refers to the mean of the loss values
    # or potentially the mean of the predicted values from the regression line.
    # Given the context "If the mean value is less than or equal to threshold Q",
    # and Q is a loss threshold, it refers to the mean of the loss history.
    mean_val = mean_y
    
    return mean_val, slope
```

**Numbered Steps:**
1.  **Initialization:** Initialize the octree using variance information from the original data [S1].
2.  **Stage Training:** Begin the first stage. Randomly crop cubic data blocks from the original data, downsample them, encode to a latent context grid, perform hierarchical sampling based on the current octree, decode, and compute the total loss (regression + PDE) [S1]. Update model weights.
3.  **Bottleneck Detection:** Starting from the $q$-th epoch, use the loss values of the last 10 epochs to perform linear regression [S1].
4.  **Termination Check:** Calculate the mean value of the linear regression model (mean loss). If the mean value is $\le Q$, terminate training immediately and save the current best model [S1].
5.  **Stage Transition Check:** If the mean value is $> Q$ AND the slope of the linear regression model is $\ge 0$, start the next stage [S1].
6.  **Octree Reconstruction:**
    *   Downsample the ground truth $D_{gt}$ by factor $a$ to get $D_{a\_gt}$ [S1].
    *   Perform inference using $D_{a\_lr}$ (downsampled $D_{a\_gt}$) and the current best model $N^k_{best}$ to get $D^k_{hr}$ [S1]. Formula: $D^k_{hr} = N^k_{best}(D_{a\_lr})$ [S1].
    *   Compute regression loss $L^k_j$ for each grid point $j$: $L^k_j = \frac{1}{c} \sum_{i=1}^{c} \alpha_i ||x_{ij} - \hat{x}^k_{ij}||_1$, where $\hat{x}^k_{ij}$ is the estimated value from $D^k_{hr}$ and $\alpha_i$ is the weight for the $i$-th physical variable [S1].
    *   Compute total regression loss $L^k_{tot} = \frac{1}{n} \sum_{j=1}^{n} L^k_j$ [S1].
    *   Recursively reconstruct the octree from the root. For each subblock, compute its total regression loss $L_{sub,k\_tot}$. Subdivide the subblock if $L_{sub,k\_tot} \ge L^k_{tot}$ AND at least one dimension of the subblock is greater than $s$ [S1].
7.  **Repeat:** Repeat the training process with the new octree until the termination condition is reached [S1].

**Edge Cases and Pitfalls:**
*   **Frequent Stage Transitions:** Frequent reconstruction is time-consuming. Mitigated by enforcing a minimum epoch count $q$ [S1].
*   **Importance Sampling Failure:** As training proceeds and weights update, the initial octree fails to represent the training data distribution, causing importance sampling to become irrelevant and gradients to converge to 0. Mitigated by reconstructing the octree at stage boundaries [S1].
*   **Inference Cost:** Full-resolution inference is time-consuming. Mitigated by downsampling the ground truth data $D_{gt}$ by a scale factor $a$ for octree reconstruction, reducing inference time to $1/a^3$ [S1].

## Operating characteristics
*   **Time Complexity:** The strategy adds overhead due to periodic octree reconstruction and inference. The reconstruction cost is proportional to the number of grid points and octree nodes. The downsampling factor $a$ is a trade-off between accuracy and speed [S1].
*   **Memory Complexity:** Requires storing the octree structure and the high-resolution output $D^k_{hr}$ during reconstruction.
*   **Strengths:** Ensures continued effectiveness of importance sampling throughout training, leading to better convergence and accuracy compared to single-stage methods [S1].
*   **Limitations:** Increased training time due to reconstruction steps. Requires careful tuning of parameters $q$ and $Q$ [S1].
*   **When to use:** When training volumetric super-resolution models with octree-based hierarchical sampling, especially when the data distribution changes significantly during training or when single-stage sampling leads to premature convergence [S1].

## Background
The Multi-stage Training Strategy was proposed in the context of octree-based hierarchical sampling optimization for the volumetric super-resolution of scientific data [S1]. It addresses the specific challenge that importance sampling, which relies on a fixed or variance-based octree, becomes ineffective as the neural network weights update and the error distribution changes [S1]. The method was validated on Rayleigh-Bénard convection problems and applied to state-of-the-art methods like MeshfreeFlowNet and TransflowNet [S1].

## Sources
[S1] Octree-based hierarchical sampling optimization for the volumetric super-resolution of scientific data, https://doi.org/10.1016/j.jcp.2024.112804, fetched 2026-10-26
