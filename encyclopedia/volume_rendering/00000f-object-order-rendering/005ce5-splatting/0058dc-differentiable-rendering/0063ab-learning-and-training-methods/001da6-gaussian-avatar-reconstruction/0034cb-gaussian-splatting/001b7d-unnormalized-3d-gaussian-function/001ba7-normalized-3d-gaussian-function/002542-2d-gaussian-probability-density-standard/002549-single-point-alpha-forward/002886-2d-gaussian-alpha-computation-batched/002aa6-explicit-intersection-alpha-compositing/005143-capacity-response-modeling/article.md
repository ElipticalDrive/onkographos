# Capacity Response Modeling

Also called: Opacity Response Gram Matrix, Quadratic Selection, Quadratic Selection via Opacity Response

## Overview
Capacity Response Modeling is a pruning method that computes individual importance scores and pairwise interaction terms for Gaussian selection by measuring RGB changes when Gaussian opacity varies [S1, unofficial]. It determines which $K_b$ members to retain in each block by accounting for joint rendering effects via the response matrix $R_b$ [S1, unofficial]. It builds on [Explicit Intersection Alpha Compositing](4814-explicit-intersection-alpha-compositing.md) by using its alpha compositing logic to derive gradients for selection, differing from [Gamma-Space Alpha Blending](7704-gamma-space-alpha-blending.md) which operates in a different color space.

## How it works
**Inputs:**
- A set of 3D Gaussians partitioned into blocks.
- Training views $v$ and their corresponding ground-truth images.
- For each block $b$, the set of Gaussians $C_b$ and their parameters (opacity, color, covariance, position).
- Target retention count $K_b$ for block $b$ (determined by a prior budget allocation step).

**Outputs:**
- A binary retention vector $x_b \in \{0, 1\}^{n_b}$ for each block, where $x_{b,i}=1$ indicates the $i$-th Gaussian is retained.

**Data Structures:**
- **Selected Pixels Set $P_b$**: A set of pairs $(v, p)$ representing view $v$ and pixel coordinate $p$.
- **Jacobian Matrix $J_b$**: Dimensions $3|P_b| \times n_b$. Rows correspond to sampled view-pixel-channel triples $(v, p, q)$ where $q \in \{1, 2, 3\}$. Columns correspond to Gaussians in block $b$.
- **Response Matrix $R_b$**: Dimensions $n_b \times n_b$. Computed as $J_b^T J_b$.

**Steps:**

1.  **Pixel Selection**:
    For each block $b$ and camera view $v$:
    a. Identify projected Gaussian centers.
    b. Augment this set with neighboring pixels obtained by offsetting each center.
    c. Remove duplicate locations and discard pixels outside the image boundaries.
    d. Collect selected pixels $p$ for view $v$ as pairs $(v, p)$ into set $P_b$ [S1, unofficial].

2.  **Rendered Color Definition**:
    Let $L_{vp}$ be the number of Gaussians covering pixel $p$ in view $v$, indexed front-to-back as $g_1, \dots, g_{L_{vp}}$.
    Let $\alpha_\ell$ be the original alpha of $g_\ell$.
    Introduce a continuous opacity multiplier $\eta_i \in [0, 1]$ for each Gaussian $i$ in the block.
    The modified alpha and incoming transmittance are:
    $$ \alpha_\ell(\eta) = \eta_{g_\ell} \alpha_\ell $$
    $$ T_\ell(\eta) = \prod_{r=1}^{\ell-1} (1 - \alpha_r(\eta)) $$
    The rendered RGB channel $q \in \{1, 2, 3\}$ at pixel $p$ in view $v$ is:
    $$ I_{vpq}(\eta) = \sum_{\ell=1}^{L_{vp}} T_\ell(\eta)\alpha_\ell(\eta)c_\ell^q + T_{L_{vp}+1}(\eta)c_{bg}^q $$
    where $c_\ell$ is the Gaussian’s view-dependent color component, $c_{bg}$ is the background color component, and $T_{L_{vp}+1}$ is the transmittance remaining after all contributors [S1, unofficial].

3.  **Jacobian Computation**:
    Differentiate each sampled RGB channel with respect to the opacity multiplier of every block member. For $i \in C_b$, the corresponding entry of $J_b \in \mathbb{R}^{3|P_b| \times n_b}$ is:
    $$ (J_b)_{(v,p,q),i} = \left. \frac{\partial I_{vpq}(\eta)}{\partial \eta_i} \right|_{\eta=1} $$
    Rows correspond to sampled view-pixel-channel triples $(v, p, q)$, and columns correspond to the Gaussians in block $b$ [S1, unofficial].

4.  **Response Matrix Construction**:
    Compute the opacity-response Gram matrix:
    $$ R_b = J_b^T J_b $$
    The diagonal score $D_i = (R_b)_{ii}$ sums the squared color derivatives for opacity of Gaussian $i$ over all selected pixels, views, and RGB channels. A larger $D_i$ indicates greater influence on the rendered image. Off-diagonal entries are signed inner products between response columns; positive entries indicate reinforcing linearized deletion effects, while negative entries indicate partial cancellation [S1, unofficial].

5.  **Quadratic Selection (QUBO)**:
    Solve the following QUBO problem for each block independently:
    $$ \min_{x_b \in \{0,1\}^{n_b}} E_b(x_b) = (1 - x_b)^T R_b (1 - x_b) + \lambda (1^T x_b - K_b)^2 $$
    Here $x_{b,i}=1$ if Gaussian is retained, $R_b$ is the response matrix, $K_b$ is the quota, and $\lambda$ is a penalty weight. The first term approximates the rendering change caused by jointly deleting the Gaussians with $x_{b,i} = 0$. The second term penalizes deviations from the allocated quota $K_b$ [S1, unofficial].

6.  **Repair and Combine**:
    Repair infeasible solver outputs if necessary. Combine block masks into a model containing exactly $K$ Gaussians [S1, unofficial].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import numpy as np

def compute_capacity_response_modeling(gaussians, views, target_counts, lambda_penalty=1.0):
    """
    gaussians: List of blocks, where each block is a list of Gaussian objects.
               Each Gaussian has attributes: opacity, color (3,), cov, pos, etc.
    views: List of view objects. Each view has: camera_params, gt_image (H, W, 3).
    target_counts: Dict mapping block_index -> K_b.
    lambda_penalty: Scalar lambda for QUBO penalty.
    
    Returns:
    retention_masks: List of boolean arrays, one per block.
    """
    retention_masks = []
    
    for b_idx, block in enumerate(gaussians):
        n_b = len(block)
        K_b = target_counts[b_idx]
        
        # Step 1: Pixel Selection
        P_b = set()
        for view in views:
            # Project centers to image plane
            projected_centers = [g.project(view.camera_params) for g in block]
            
            # Augment with neighbors (e.g., 3x3 grid around each center)
            selected_pixels = set()
            for center in projected_centers:
                x, y = center
                for dy in [-1, 0, 1]:
                    for dx in [-1, 0, 1]:
                        ny, nx = int(y) + dy, int(x) + dx
                        if 0 <= ny < view.gt_image.shape[0] and 0 <= nx < view.gt_image.shape[1]:
                            selected_pixels.add((ny, nx))
            
            # Add to global set for block
            for p in selected_pixels:
                P_b.add((view.id, p))
        
        if not P_b:
            # Edge case: No valid pixels, retain all or none based on strategy
            retention_masks.append(np.ones(n_b, dtype=bool))
            continue

        # Step 2 & 3: Jacobian Computation
        # J_b shape: (3 * |P_b|, n_b)
        num_pixels = len(P_b)
        J_b = np.zeros((3 * num_pixels, n_b))
        
        pixel_list = list(P_b)
        
        for row_idx, (v_id, p) in enumerate(pixel_list):
            # Find view object
            view = get_view_by_id(views, v_id)
            gt_color = view.gt_image[p[0], p[1]] # H, W, C -> p is (y, x)
            
            # Get Gaussians covering this pixel in front-to-back order
            covering_gaussians = get_covering_gaussians(block, view, p)
            L_vp = len(covering_gaussians)
            
            # Map global Gaussian index in block to local index in covering list
            # Note: block contains all Gaussians in the block, covering_gaussians is a subset
            # We need to compute derivative w.r.t each Gaussian in the BLOCK, not just covering ones.
            # However, if a Gaussian does not cover the pixel, its derivative is 0.
            
            for col_idx, g_global in enumerate(block):
                # Check if g_global is in covering_gaussians
                if g_global in covering_gaussians:
                    # Find its position in the front-to-back list
                    local_idx = covering_gaussians.index(g_global)
                    
                    # Compute derivative dI/deta_i at eta=1
                    # Using chain rule on alpha_composite
                    deriv = compute_alpha_derivative(g_global, covering_gaussians, local_idx, view)
                    J_b[row_idx * 3, col_idx] = deriv[0] # R channel
                    J_b[row_idx * 3 + 1, col_idx] = deriv[1] # G channel
                    J_b[row_idx * 3 + 2, col_idx] = deriv[2] # B channel
                else:
                    J_b[row_idx * 3 : row_idx * 3 + 3, col_idx] = 0.0

        # Step 4: Response Matrix
        R_b = J_b.T @ J_b
        
        # Step 5: QUBO Solution
        # Approximate solution using greedy selection or simple heuristic if solver not available
        # For implementation completeness, we use a greedy approach based on diagonal scores
        # and interaction penalties, as exact QUBO solvers are external.
        
        # Diagonal scores
        diag_scores = np.diag(R_b)
        
        # Greedy selection heuristic:
        # Initialize all as deleted (0), then iteratively add best candidates until K_b reached
        # Or initialize all as kept (1) and remove worst.
        # The objective is min (1-x)^T R (1-x) + lambda (1^T x - K)^2
        # Let y = 1-x be deletion mask. Min y^T R y + lambda (n - sum(y) - K)^2
        # subject to sum(y) = n - K_b
        
        num_to_delete = n_b - K_b
        if num_to_delete <= 0:
            retention_masks.append(np.ones(n_b, dtype=bool))
            continue
            
        # Simple greedy deletion:
        # Start with all kept. In each step, remove the Gaussian that minimizes the increase in error.
        # Error change for removing i: approx 2 * R[i, kept] - R[i,i] ? 
        # Exact gradient of y^T R y w.r.t y_i is 2 (R y)_i.
        # Since we are setting y_i from 0 to 1, the marginal cost is complex due to quadratic term.
        # A simpler approximation often used: select K_b Gaussians with highest diagonal scores,
        # then adjust for interactions.
        
        # Given the constraint of "no placeholders", we implement a basic greedy selection
        # that respects the quadratic term approximately.
        
        # Initialize x as all 1s
        x = np.ones(n_b, dtype=float)
        
        # We need to remove num_to_delete items.
        # Calculate current energy
        def energy(x_vec):
            y = 1 - x_vec
            term1 = y @ R_b @ y
            term2 = lambda_penalty * (np.sum(x_vec) - K_b)**2
            return term1 + term2
            
        current_energy = energy(x)
        
        for _ in range(num_to_delete):
            best_i = -1
            best_energy = np.inf
            
            for i in range(n_b):
                if x[i] == 0: continue # Already deleted
                
                # Try deleting i
                x_trial = x.copy()
                x_trial[i] = 0
                e = energy(x_trial)
                
                if e < best_energy:
                    best_energy = e
                    best_i = i
            
            if best_i != -1:
                x[best_i] = 0
            else:
                break # Should not happen if num_to_delete <= n_b
        
        retention_masks.append(x.astype(bool))
        
    return retention_masks

def compute_alpha_derivative(g, covering_list, local_idx, view):
    """
    Computes dI/deta_g at eta=1.
    I = sum_{l=1}^{L} T_l alpha_l c_l + T_{L+1} c_bg
    alpha_l(eta) = eta_l alpha_l
    T_l(eta) = prod_{r=1}^{l-1} (1 - alpha_r(eta))
    
    dI/deta_g = sum_{l=1}^{L} [ dT_l/deta_g * alpha_l c_l + T_l * dalpha_l/deta_g * c_l ] + dT_{L+1}/deta_g * c_bg
    
    dalpha_l/deta_g = alpha_l if l==g else 0
    dT_l/deta_g = -alpha_g * T_l / (1 - alpha_g) if g < l else 0
    
    At eta=1:
    alpha_l is original alpha.
    T_l is original transmittance.
    """
    L = len(covering_list)
    c_bg = view.background_color # Assumed available
    
    # Precompute original values
    alphas = np.array([g.alpha for g in covering_list])
    colors = np.array([g.color for g in covering_list]) # (L, 3)
    Ts = np.zeros(L + 1)
    Ts[0] = 1.0
    for l in range(1, L + 1):
        Ts[l] = Ts[l-1] * (1 - alphas[l-1])
        
    # Derivative calculation
    dI = np.zeros(3)
    
    # Term 1: Sum over l
    for l in range(1, L + 1):
        # dalpha_l/deta_g
        if l == local_idx + 1: # local_idx is 0-based, l is 1-based
            d_alpha = alphas[l-1]
        else:
            d_alpha = 0.0
            
        # dT_l/deta_g
        if local_idx + 1 < l: # g < l
            # T_l = T_g * (1-alpha_g) * ... * (1-alpha_{l-1})
            # dT_l/deta_g = -alpha_g * T_l / (1-alpha_g)
            # Note: T_l in formula is product up to l-1.
            # If g < l, then alpha_g is in the product.
            # T_l(eta) = (1-alpha_g(eta)) * ...
            # d/deta_g (1-alpha_g) = -alpha_g
            # So dT_l/deta_g = -alpha_g * T_l / (1-alpha_g)
            d_T = -alphas[local_idx] * Ts[l-1] / (1 - alphas[local_idx] + 1e-8)
        else:
            d_T = 0.0
            
        dI += d_T * alphas[l-1] * colors[l-1] + Ts[l-1] * d_alpha * colors[l-1]
        
    # Term 2: Background transmittance T_{L+1}
    # T_{L+1} = prod_{r=1}^L (1-alpha_r)
    if local_idx + 1 <= L:
        d_T_bg = -alphas[local_idx] * Ts[L] / (1 - alphas[local_idx] + 1e-8)
    else:
        d_T_bg = 0.0
        
    dI += d_T_bg * c_bg
    
    return dI
```

## Operating characteristics
- **Time Complexity**: Dominated by the Jacobian computation $O(|P_b| \cdot L_{vp} \cdot n_b)$ and the QUBO solve. Greedy selection is $O(n_b^2)$ per block.
- **Memory Complexity**: $O(|P_b| \cdot n_b)$ for the Jacobian and $O(n_b^2)$ for the response matrix $R_b$.
- **Strengths**: Accounts for pairwise interactions between Gaussians, leading to better visual quality than independent pruning.
- **Limitations**: Requires computing derivatives for all selected pixels, which can be memory-intensive for large blocks. The QUBO problem is NP-hard, so approximations are used.
- **When to use**: When pruning 3D Gaussian Splatting models while preserving visual fidelity and handling occlusion/interaction effects.
- **When not to use**: When real-time performance is critical and pruning overhead is prohibitive, or when interactions are negligible.

## Background
Originated in "SplatQ: Interaction-Aware Pruning of 3D Gaussian Splatting via Quadratic Selection" (2026) [S1, unofficial]. It extends standard pruning by incorporating a quadratic selection mechanism based on the opacity response Gram matrix.

## Sources
[S1] SplatQ: Interaction-Aware Pruning of 3D Gaussian Splatting via Quadratic Selection, https://arxiv.org/abs/2610.13503, fetched 2026-10-29 (unofficial)
