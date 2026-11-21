# Neural SDF Inference via Gaussian Pulling

Also called: Center Loss, Disk Loss, Neural Signed Distance Function Inference through Splatting 3D Gaussians Pulled on Zero-Level Set, Neural-Pull, Pulling to Centers, Pulling to Disks

## Overview
Neural SDF Inference via Gaussian Pulling is a method for reconstructing continuous 3D surfaces from multi-view RGB images by jointly optimizing a set of 3D Gaussians and a neural Signed Distance Function (SDF) [S1]. It solves the problem of inferring accurate geometry from 3D Gaussian Splatting (3DGS), which typically suffers from discreteness, sparseness, and off-surface drift, making direct SDF inference difficult [S1]. Unlike standard 3DGS which renders appearance without explicit surface constraints, this method dynamically aligns Gaussians to the SDF's zero-level set and constrains them to be thin disks tangent to the surface, enabling robust surface reconstruction [S1]. It differs from [Gaussian Representation and Truncation](12855-gaussian-representation-and-truncation.md) by explicitly coupling the Gaussian positions to a continuous implicit field via differentiable pulling, and from [Mesh Extraction and Representation](14095-mesh-extraction-and-representation.md) by learning the SDF directly rather than extracting meshes from static Gaussian clusters.

## How it works
**Inputs:** Posed multi-view RGB images $\{v_i\}_{i=1}^I$.
**Outputs:** A neural SDF network $f$ and a set of optimized 3D Gaussians $\{g_j\}_{j=1}^J$.

**Data Structures:**
1.  **3D Gaussians:** Each Gaussian $g_j$ is defined by mean $\boldsymbol{\mu}_j$, covariance matrix $\boldsymbol{\Sigma}_j$ (derived from rotation $\mathbf{R}_j$ and scale $\mathbf{S}_j$), opacity $\alpha_j$, and view-dependent color $\mathbf{c}_j$. The probability density at query $\mathbf{q}$ is:
    $$ p_j(\mathbf{q}) = \exp\left(-\frac{1}{2}(\mathbf{q} - \boldsymbol{\mu}_j)^\top \boldsymbol{\Sigma}_j^{-1} (\mathbf{q} - \boldsymbol{\mu}_j)\right) $$
2.  **Neural SDF:** An MLP with 8 layers, 256 hidden units, and ReLU activation, initialized to represent a sphere [S1]. It predicts signed distance $s = f(\mathbf{q})$ and gradient $\nabla f(\mathbf{q})$ at any point $\mathbf{q}$.

**Numbered Steps:**

1.  **Initialization:** Initialize $J$ 3D Gaussians $\{g_j\}$ and the SDF network $f$.
2.  **Gaussian Rendering:** Render the Gaussians $\{g_j\}$ into RGB images using volume rendering:
    $$ C'(u, v) = \sum_{j=1}^{J} c_j \alpha_j p_j(u, v) \prod_{k=1}^{j-1} (1 - \alpha_k p_k(u, v)) $$
    where $p_j(u, v)$ is the 2D projection probability of the $j$-th Gaussian at pixel $(u, v)$ [S1].
3.  **Neural Pulling (Gaussian Alignment):** For each Gaussian $g_j$, compute the signed distance $s_j = f(\boldsymbol{\mu}_j)$ and gradient $\nabla f(\boldsymbol{\mu}_j)$. Pull the Gaussian center to the zero-level set to obtain $\boldsymbol{\mu}'_j$:
    $$ \boldsymbol{\mu}'_j = \boldsymbol{\mu}_j - s_j \frac{\nabla f(\boldsymbol{\mu}_j)}{\|\nabla f(\boldsymbol{\mu}_j)\|} $$
    The pulled Gaussian $g'_j$ shares attributes with $g_j$ except for the center $\boldsymbol{\mu}'_j$ [S1].
4.  **Pulled Gaussian Rendering:** Render the pulled Gaussians $\{g'_j\}$ into images $\{v'_i\}$ using the same volume rendering equation [S1].
5.  **SDF Inference (Query Pulling):** For query points $\mathbf{q}$ in the neighborhood of the surface, pull them onto the zero-level set using the SDF gradient:
    $$ \mathbf{q}' = \mathbf{q} - f(\mathbf{q}) \frac{\nabla f(\mathbf{q})}{\|\nabla f(\mathbf{q})\|} $$
    Identify the nearest pulled Gaussian $\bar{g}$ with center $\bar{\boldsymbol{\mu}}$ based on distance $\|\boldsymbol{\mu}'_j - \mathbf{q}\|^2$ [S1].
6.  **Loss Computation:** Compute the total loss comprising rendering, thin disk, tangent, pulling, and orthogonal constraints (see Formulas).
7.  **Optimization:** Update $\{g_j\}$ and $f$ parameters using gradient descent to minimize the total loss [S1].
8.  **Mesh Extraction:** After optimization, extract the mesh from the learned SDF $f$ using the marching cubes algorithm. For small scenes, use resolution 800; for large scenes, split into parts and reconstruct each with resolution 800 [S1].

**Formulas:**

*   **Thin Disk Constraint ($L_{Thin}$):** Encourages Gaussians to become flat disks by penalizing the smallest variance. The normal $\mathbf{n}_j$ is defined by the axis with minimum variance $\bar{r} = \min\{r_1, r_2, r_3\}$ [S1].
    $$ L_{Thin} = \|\min\{r_1, r_2, r_3\}\|_1 $$
*   **Tangent Constraint ($L_{Tangent}$):** Aligns the Gaussian normal $\mathbf{n}_j$ with the SDF gradient at the pulled center $\boldsymbol{\mu}'_j$:
    $$ L_{Tangent} = 1 - \frac{\nabla f(\boldsymbol{\mu}'_j)}{\|\nabla f(\boldsymbol{\mu}'_j)\|} \cdot \mathbf{n}_j $$
*   **Pulling Probability Loss ($L_{Pull}$):** Maximizes the probability that the pulled query $\mathbf{q}'$ belongs to the nearest pulled Gaussian $\bar{g}$. Implemented as the negative logarithm of the probability:
    $$ L_{Pull}(\mathbf{q}'; \bar{\boldsymbol{\mu}}) = -\log \left( \exp\left(-\frac{1}{2}(\mathbf{q}' - \bar{\boldsymbol{\mu}})^\top \bar{\boldsymbol{\Sigma}}^{-1} (\mathbf{q}' - \bar{\boldsymbol{\mu}})\right) \right) $$
    where $\bar{\boldsymbol{\Sigma}}$ is the covariance of $\bar{g}$ [S1].
*   **Orthogonal Constraint ($L_{Orthogonal}$):** Aligns the SDF gradient at query $\mathbf{q}$ with the normal $\bar{\mathbf{n}}$ of the nearest pulled Gaussian disk:
    $$ L_{Orthogonal} = 1 - \frac{\nabla f(\mathbf{q})}{\|\nabla f(\mathbf{q})\|} \cdot \bar{\mathbf{n}} $$
*   **Rendering Loss ($L_{Splatting}$):** Combines L1 and D-SSIM terms between pulled rendered images $\{v'_i\}$ and ground truth $\{v_i\}$:
    $$ L_{Splatting} = 0.8 \cdot L_1(v'_i, v_i) + 0.2 \cdot L_{D-SSIM}(v'_i, v_i) $$
*   **Total Objective:**
    $$ \min_{\{g_j\}, f} L_{Splatting} + \alpha L_{Thin} + \beta L_{Tangent} + \gamma L_{Pull} + \delta L_{Orthogonal} $$

**Parameters:**
*   Regularization weights: $\alpha=100$, $\beta=0.1$, $\gamma=1$, $\delta=0.1$ [S1].
*   Optimization iterations: 15k total. Densification stops and pulling/constraints are incorporated at 7k iterations [S1].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def train_neural_sdf_gaussian_pulling(images, initial_gaussians, sdf_network):
    """
    Jointly optimize 3D Gaussians and Neural SDF using Gaussian Pulling.
    """
    # Hyperparameters
    alpha = 100.0  # Thin disk weight
    beta = 0.1     # Tangent weight
    gamma = 1.0    # Pulling probability weight
    delta = 0.1    # Orthogonal weight
    total_iters = 15000
    pull_start_iter = 7000
    
    # Optimizers for Gaussians and SDF
    opt_gaussians = Adam(initial_gaussians.parameters())
    opt_sdf = Adam(sdf_network.parameters())
    
    for iteration in range(1, total_iters + 1):
        # 1. Sample a batch of camera views and corresponding GT images
        batch_views, gt_images = sample_batch(images)
        
        # 2. Render current Gaussians (optional for densification before pull_start_iter)
        # Note: Densification logic is handled externally or via standard 3DGS rules
        # until iteration 7000.
        
        # 3. Neural Pulling: Align Gaussians to SDF Zero-Level Set
        pulled_gaussians = []
        for g in initial_gaussians:
            mu = g.mean
            s = sdf_network(mu)  # Signed distance f(mu)
            grad = sdf_network.gradient(mu)  # Gradient df/dmu
            
            # Normalize gradient
            grad_norm = torch.norm(grad)
            if grad_norm > 1e-8:
                grad_normalized = grad / grad_norm
            else:
                grad_normalized = grad # Fallback for degenerate case [G]
            
            # Pull center to zero-level set
            mu_pulled = mu - s * grad_normalized
            
            # Create pulled Gaussian with same attributes but new center
            g_pulled = Gaussian(mean=mu_pulled, cov=g.cov, alpha=g.alpha, color=g.color)
            pulled_gaussians.append(g_pulled)
            
        # 4. Render Pulled Gaussians
        rendered_images = render_volume_rendering(pulled_gaussians, batch_views)
        
        # 5. Compute Rendering Loss
        l1_loss = compute_l1(rendered_images, gt_images)
        ssim_loss = compute_d_ssim(rendered_images, gt_images)
        l_splatting = 0.8 * l1_loss + 0.2 * ssim_loss
        
        total_loss = l_splatting
        
        # 6. Geometry Constraints (Active after pull_start_iter)
        if iteration >= pull_start_iter:
            # Thin Disk Loss
            l_thin = 0.0
            for g in initial_gaussians:
                variances = g.cov.diagonal() # Extract r1, r2, r3
                min_var = torch.min(variances)
                l_thin += torch.abs(min_var)
            l_thin = torch.mean(l_thin)
            total_loss += alpha * l_thin
            
            # Tangent Loss
            l_tangent = 0.0
            for g, g_pulled in zip(initial_gaussians, pulled_gaussians):
                mu_p = g_pulled.mean
                grad_p = sdf_network.gradient(mu_p)
                grad_norm_p = torch.norm(grad_p)
                
                # Determine Gaussian normal from min variance axis
                variances = g.cov.diagonal()
                min_idx = torch.argmin(variances)
                # Get axis direction corresponding to min variance
                # Assuming covariance is diagonal in local frame, transform to world
                # For simplicity, assume g.normal is stored or derived from rotation
                n_g = g.normal 
                
                if grad_norm_p > 1e-8:
                    grad_norm_p_normalized = grad_p / grad_norm_p
                else:
                    grad_norm_p_normalized = grad_p # Fallback [G]
                    
                l_tangent += 1.0 - torch.dot(grad_norm_p_normalized, n_g)
            l_tangent = torch.mean(l_tangent)
            total_loss += beta * l_tangent
            
            # Pulling Probability and Orthogonal Loss (on query points)
            # Sample query points q near the surface
            query_points = sample_queries_near_surface(pulled_gaussians)
            l_pull = 0.0
            l_orthogonal = 0.0
            
            for q in query_points:
                s_q = sdf_network(q)
                grad_q = sdf_network.gradient(q)
                grad_norm_q = torch.norm(grad_q)
                
                if grad_norm_q > 1e-8:
                    grad_norm_q_normalized = grad_q / grad_norm_q
                else:
                    grad_norm_q_normalized = grad_q # Fallback [G]
                
                # Pull query to zero-level set
                q_pulled = q - s_q * grad_norm_q_normalized
                
                # Find nearest pulled Gaussian
                nearest_g_pulled = find_nearest_gaussian(q_pulled, pulled_gaussians)
                mu_bar = nearest_g_pulled.mean
                cov_bar = nearest_g_pulled.cov
                
                # Pulling Probability Loss (Negative Log Likelihood)
                diff = q_pulled - mu_bar
                # Mahalanobis distance squared
                dist_sq = torch.dot(diff, torch.inverse(cov_bar) @ diff)
                prob = torch.exp(-0.5 * dist_sq)
                l_pull -= torch.log(prob + 1e-8) # Avoid log(0)
                
                # Orthogonal Loss
                n_bar = nearest_g_pulled.normal
                l_orthogonal += 1.0 - torch.dot(grad_norm_q_normalized, n_bar)
                
            l_pull = torch.mean(l_pull)
            l_orthogonal = torch.mean(l_orthogonal)
            total_loss += gamma * l_pull + delta * l_orthogonal
            
        # 7. Backpropagation
        opt_gaussians.zero_grad()
        opt_sdf.zero_grad()
        total_loss.backward()
        opt_gaussians.step()
        opt_sdf.step()
        
    return initial_gaussians, sdf_network
```

## Subtopics

### Volume Rendering with Surface Patch Sensing
A variant that infers SDFs by defining a local patch around the estimated ray-surface intersection using the SDF, allowing direct application of surface constraints [S2]. It samples points around the patch using a Gaussian distribution, applies mask and weight filtering, and pulls points to the zero-level set using gradients [S2]. It outperforms NeuralRGBD, MonoSDF, DROID-SLAM, NICER-SLAM, and NICE-SLAM on ScanNet, Replica, and Tanks and Temples, especially on planar structures [S2]. It uses ground truth depth maps for ScanNet and monocular cues from Omnidata for Replica [S2].

## Operating characteristics
**Time and Memory Complexity:** The method requires joint optimization of Gaussians and an MLP-based SDF. Training time is comparable to 2DGS (21.8m vs 20.5m on DTU) and significantly faster than NeuS (~12h vs 37.6m on Tanks and Temples) [S1]. Memory usage is dominated by the Gaussian attributes and the SDF network parameters, with mesh extraction requiring resolution 800 grids [S1].

**Measured Performance:** On the DTU dataset, it achieves a mean Chamfer Distance of 0.52, outperforming 2DGS (0.54) and GSurfels (0.56) [S1]. On Tanks and Temples, it achieves a mean F-score of 0.84, surpassing NeuS (0.54) [S1].

**Strengths and Limitations:** Strengths include robust surface reconstruction from sparse and non-uniformly distributed Gaussians, avoidance of background artifacts (e.g., large sky spheres) common in TSDF or Poisson fusion, and high rendering efficiency [S1]. Limitations include sensitivity to the initialization of the SDF (initialized as a sphere) and the need for multi-view consistency; it may struggle with highly dynamic scenes or those lacking sufficient overlap [S1].

**When to Use:** Use when accurate, smooth, and complete surface reconstruction is required from multi-view RGB images, especially when avoiding background artifacts is critical. It is suitable for static scenes with moderate to high multi-view coverage.

## Background
The method was proposed by Zhang et al. in 2024, building on the principles of 3D Gaussian Splatting and Neural Pulling [S1]. It addresses the limitations of previous methods that either added volume rendering branches or used monocular priors, by seamlessly merging 3DGS with neural SDF learning using multi-view consistency [S1]. The "pulling to disks" strategy was introduced to overcome the overfitting issues of "pulling to centers," where the loss gradient remains constant as the query approaches the center, leading to poor generalization on sparse data [S1].

## Sources
[S1] Neural Signed Distance Function Inference through Splatting 3D Gaussians Pulled on Zero-Level Set, https://doi.org/10.48550/arXiv.2410.14189, fetched 2026-10-29
[S2] Sensing Surface Patches in Volume Rendering for Inferring Signed Distance Functions, https://doi.org/10.1609/aaai.v39i4.32426, fetched 2026-10-26
