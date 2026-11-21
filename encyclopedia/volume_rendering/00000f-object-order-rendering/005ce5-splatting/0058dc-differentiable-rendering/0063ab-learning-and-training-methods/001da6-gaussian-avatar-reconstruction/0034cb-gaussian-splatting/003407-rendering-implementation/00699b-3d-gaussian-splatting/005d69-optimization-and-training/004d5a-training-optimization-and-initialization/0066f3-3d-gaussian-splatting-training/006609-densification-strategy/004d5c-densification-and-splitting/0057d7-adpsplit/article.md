# AdpSplit

Also called: Adaptive Split, Error-Driven Adaptive Splitting

## Overview
AdpSplit is an error-driven adaptive split operator for 3D Gaussian Splatting that replaces fixed-cardinality random child placement with view-loss-aware child generation [S1, unofficial]. It determines the number of split children and initializes their parameters from L1-pixel-error region statistics, enabling fewer densification iterations and reduced training time while preserving rendering quality [S1, unofficial]. Unlike vanilla 3DGS which uses a fixed $N=2$ binary split, AdpSplit adaptively determines the number of children based on the observed error structure, avoiding the bottleneck where binary splits require $\log_2 G$ rounds to expose a structure requiring $G$ Gaussians [S1, unofficial].

## How it works
**Inputs and Outputs:**
*   **Input:** A set of split candidates $\mathcal{S}$ identified by the standard 3DGS gradient-and-size rule; the current Gaussian set $\mathcal{G}$; a set of $V$ sampled training views.
*   **Output:** A refined Gaussian set $\mathcal{G}'$ with split candidates replaced by their children and a preserved parent copy.

**Data Structures:**
*   **Gaussian $g_i$:** Defined by mean $\mu_i \in \mathbb{R}^3$, covariance $\Sigma_i \in \mathbb{R}^{3 \times 3}$ (parameterized by scale $s_i$ and rotation $q_i$), opacity $\alpha_i$, and spherical harmonic coefficients $\mathbf{c}_i$.
*   **Dominant Gaussian Map $D_v$:** For each pixel $u$ in view $v$, stores the index of the Gaussian with the largest alpha-weighted contribution.
*   **Error Region $R_{qv}$:** A set of 2D pixel coordinates $\{u\}$ sharing the same error band, dominated by candidate $i$, and connected via 8-neighbor chains.

**Parameters:**
*   $\tau_{l1}$: Normalized L1-error threshold. Default: 0.1 [S1, unofficial].
*   $r_{erode}$: Erosion filter length. Default: 2 [S1, unofficial].
*   $L$: Number of error-level bands. Default: 3 [S1, unofficial].
*   $m_{min}$: Minimum error-region area threshold. Default: 5 [S1, unofficial].
*   $\gamma_d$: Distance threshold for merging. Default: 2.0 [S1, unofficial].
*   $\gamma_c$: Color threshold for merging. Default: 0.15 [S1, unofficial].
*   $N_{max}$: Maximum children per parent. Default: 19 (9 for outdoor MipNeRF360/Deep-Blending playroom) [S1, unofficial].
*   $V$: Number of sampled views. Default: 20 (10 for outdoor MipNeRF360/Deep-Blending playroom) [S1, unofficial].
*   $\epsilon$: Small constant for numerical stability in Mahalanobis distance. Value not specified [S1, unofficial].

**Steps:**

1.  **Candidate Selection:** Identify split candidates $\mathcal{S}$ using the standard 3DGS rule: $i \in \mathcal{S} \iff g_i \ge \tau_g \land \max(s_i) > \tau_s$, where $g_i$ is the average gradient statistic and $\tau_g, \tau_s$ are densification thresholds [S1, unofficial].

2.  **Error Region Partitioning:**
    For each candidate $i \in \mathcal{S}$:
    a.  Render $V$ sampled training views. Collect the dominant Gaussian map $D_v$ for each view $v$:
        $$ D_v(u) = \arg \max_{k \in \mathcal{O}_v(u)} T_{k,v}(u) \alpha_{k,v}(u) $$
        where $\mathcal{O}_v(u)$ is the set of Gaussians contributing to pixel $u$, and $T_{k,v}(u)$ is the transmittance [S1, unofficial].
    b.  Compute the min-max-normalized per-pixel L1 error map $E_v(u)$:
        $$ E_v(u) = \frac{|\hat{I}_v(u) - I_{gt,v}(u)|_1 - E_{min}}{E_{max} - E_{min}} $$
        where $E_{min}$ and $E_{max}$ are the minimum and maximum L1 errors over all pixels in view $v$ [S1, unofficial].
    c.  Create a binary metric map $M_v(u) = \mathbb{I}[E_v(u) > \tau_{l1}]$ marking high-error pixels [S1, unofficial].
    d.  Apply an erosion filter with length $r_{erode}$ to $M_v$ to remove isolated noisy pixels [S1, unofficial].
    e.  Stratify normalized error values into $L$ error-level bands over $[\tau_{l1}, 1]$, denoting the band index by $B_v(u)$ [S1, unofficial].
    f.  Identify distinct error regions $R_{qv}$ for candidate $i$ in view $v$ as maximal groups of neighboring high-error pixels dominated by $i$ and sharing the same error band:
        $$ R_{qv} = \{ u \mid M_v(u) = 1, D_v(u) = i, u \sim_8 u_q, B_v(u) = B_v(u_q) \} $$
        where $u \sim_8 u_q$ means $u$ is reachable from seed $u_q$ via an 8-neighbor chain satisfying the same conditions [S1, unofficial].
    g.  Discard regions with area $A_{qv} < m_{min}$ [S1, unofficial].
    h.  For each retained region $R_{qv}$, compute statistics: centroid $(\bar{x}_{qv}, \bar{y}_{qv})$, area $A_{qv}$, dominant PCA direction $e_{q1,v}$, orthogonal direction $e_{q2,v}$, standard deviations $(\sigma_{1,v}, \sigma_{2,v})$ from pixel covariance, and ground-truth RGB value $r_{qv}$ at the centroid [S1, unofficial].

3.  **Child Initialization:**
    For each retained region $R_{qv}$ of candidate $i$:
    a.  Define the camera ray $\ell_{qv}(t) = o_v + t \hat{d}_{qv}$, where $o_v$ is the camera center and $\hat{d}_{qv}$ is the unit ray direction [S1, unofficial].
    b.  Compute the optimal depth $t^*_{qv}$ by minimizing the Mahalanobis distance to the parent center $\mu_i$:
        $$ t^*_{qv} = \arg \min_t (\ell_{qv}(t) - \mu_i)^\top \Sigma_i^{-1} (\ell_{qv}(t) - \mu_i) = \frac{(\mu_i - o_v)^\top \Sigma_i^{-1} \hat{d}_{qv}}{\hat{d}_{qv}^\top \Sigma_i^{-1} \hat{d}_{qv} + \epsilon} $$
    c.  Initialize the child parameters:
        $$ \mu_{qv} = \ell_{qv}(t^*_{qv}) $$
        $$ R_{qv} = [\hat{u}_1 \mid \hat{u}_2 \mid z_v] $$
        $$ s_{qv} = (\|\mathbf{a}_{q1,v}\|, \|\mathbf{a}_{q2,v}\|, \|\mathbf{a}_{q2,v}\|) $$
        $$ c_{qv} = r_{qv}, \quad \alpha_{qv} = \alpha_i $$
        where $\hat{u}_1, \hat{u}_2$ are orthonormal in-plane axes from unprojected PCA axes $\mathbf{a}_{q1,v}, \mathbf{a}_{q2,v}$, and $z_v$ is the camera forward direction [S1, unofficial].

4.  **Cross-View Child Merging:**
    For each parent $i$, merge redundant children generated from different views:
    a.  Connect two child proposals $G^{(j)}$ and $G^{(k)}$ if their symmetric Mahalanobis distance is below $\gamma_d$ and their maximum RGB-channel difference is below $\gamma_c$:
        $$ \delta^\top \Sigma_j^{-1} \delta + \delta^\top \Sigma_k^{-1} \delta \le \gamma_d \quad \text{and} \quad \|c^{(j)} - c^{(k)}\|_\infty \le \gamma_c $$
        where $\delta = \mu^{(k)} - \mu^{(j)}$ [S1, unofficial].
    b.  Group connected proposals in the merge graph (transitive closure). For a merged group $\mathcal{M}$ with $M = |\mathcal{M}|$ proposals:
        $$ \mu_{\mathcal{M}} = \frac{1}{M} \sum_{m \in \mathcal{M}} \mu^{(m)}, \quad c_{\mathcal{M}} = \frac{1}{M} \sum_{m \in \mathcal{M}} c^{(m)} $$
        $$ \bar{\Sigma}_{\mathcal{M}} = \frac{1}{M} \sum_{m \in \mathcal{M}} \Sigma^{(m)} = E \Lambda E^\top $$
        $$ \lambda_{\mathcal{M},r} = \max_{m \in \mathcal{M}} \left( |e_r^\top (\mu^{(m)} - \mu_{\mathcal{M}})| + \sigma_{m,r} \right)^2, \quad r \in \{1, 2, 3\} $$
        $$ \Sigma_{\mathcal{M}} = E \text{diag}(\lambda_{\mathcal{M}}) E^\top $$
        where $e_r$ are eigenvectors of $\bar{\Sigma}_{\mathcal{M}}$ and $\sigma_{m,r}$ is the $1\sigma$ extent of member $m$ along $e_r$ [S1, unofficial].
    c.  Sort children by covariance extent and keep the largest $N_{max}$ proposals [S1, unofficial].

5.  **Insertion:**
    a.  Append the merged children to the Gaussian set.
    b.  Include a parent copy with opacity reduced by a factor of $N_i + 1$ (where $N_i$ is the number of children) and scale unchanged to cover error-perfect regions and keep total opacity unchanged [S1, unofficial].
    c.  Remove the original parent candidate from the set [S1, unofficial].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import torch
import numpy as np

def adp_split(candidates, gaussians, views, params):
    """
    candidates: List of Gaussian indices to split.
    gaussians: Current set of Gaussians.
    views: List of view indices to sample.
    params: Dictionary of hyperparameters.
    Returns: Updated list of Gaussians.
    """
    new_gaussians = []
    for i in candidates:
        parent = gaussians[i]
        children = []
        
        # 1. Error Region Partitioning
        regions = []
        for v in views[:params['V']]:
            # Render view v and get dominant map D_v and error map E_v
            D_v, E_v = render_view(v, gaussians)
            
            # Normalize error
            E_min = E_v.min()
            E_max = E_v.max()
            if E_max == E_min:
                continue
            E_norm = (E_v - E_min) / (E_max - E_min)
            
            # Binary mask and erosion
            M_v = (E_norm > params['tau_l1']).float()
            M_v = erosion_filter(M_v, params['r_erode'])
            
            # Stratify into bands
            # Assuming L bands over [tau_l1, 1] in normalized space
            # This step requires binning E_norm values > tau_l1
            bands = stratify_error(E_norm, M_v, params['L'])
            
            # Find regions dominated by parent i
            # D_v contains Gaussian indices
            parent_mask = (D_v == i)
            active_mask = M_v * parent_mask * (bands > 0) # bands=0 if not high error
            
            # Group connected components (8-neighbor) with same band
            # This is typically done via connected components or flood fill
            region_indices = find_connected_regions(active_mask, bands)
            
            for q, region_idx in enumerate(region_indices):
                region_pixels = get_region_pixels(region_idx, active_mask)
                if len(region_pixels) < params['m_min']:
                    continue
                
                # Compute statistics
                centroid = compute_centroid(region_pixels)
                area = len(region_pixels)
                pca_axes, stds = compute_pca(region_pixels)
                gt_color = compute_gt_color(centroid, v)
                
                regions.append({
                    'centroid': centroid,
                    'area': area,
                    'pca_axes': pca_axes,
                    'stds': stds,
                    'gt_color': gt_color,
                    'view': v
                })
        
        # 2. Child Initialization
        child_proposals = []
        for reg in regions:
            # Back-project centroid
            o_v, d_hat = get_camera_ray(reg['view'], reg['centroid'])
            
            # Mahalanobis distance minimization
            # t* = ((mu_i - o_v)^T Sigma_i^-1 d_hat) / (d_hat^T Sigma_i^-1 d_hat + eps)
            diff = parent['mu'] - o_v
            inv_Sigma = torch.linalg.inv(parent['Sigma'])
            numerator = torch.dot(diff, torch.matmul(inv_Sigma, d_hat))
            denominator = torch.dot(d_hat, torch.matmul(inv_Sigma, d_hat)) + 1e-6
            t_star = numerator / denominator
            
            mu_child = o_v + t_star * d_hat
            
            # Initialize rotation and scale
            # u1, u2 from unprojected PCA axes
            # z_v is camera forward direction
            u1, u2 = unproject_pca_axes(reg['pca_axes'], reg['view'])
            z_v = get_camera_forward(reg['view'])
            
            R_child = torch.stack([u1, u2, z_v], dim=1)
            s_child = torch.tensor([
                torch.norm(reg['pca_axes'][0]),
                torch.norm(reg['pca_axes'][1]),
                torch.norm(reg['pca_axes'][1]) # Thickness along camera forward
            ])
            
            child_proposals.append({
                'mu': mu_child,
                'R': R_child,
                's': s_child,
                'c': reg['gt_color'],
                'alpha': parent['alpha'],
                'Sigma': compute_covariance(R_child, s_child)
            })
        
        # Fallback if no regions found
        if not child_proposals:
            # Vanilla split with N=2
            children = vanilla_split(parent)
        else:
            # 3. Cross-View Child Merging
            merged_children = merge_children(child_proposals, params['gamma_d'], params['gamma_c'])
            
            # Sort by covariance extent and keep N_max
            merged_children.sort(key=lambda c: torch.trace(c['Sigma']), reverse=True)
            children = merged_children[:params['N_max']]
            
            # 4. Insertion
            # Add parent copy with reduced opacity
            parent_copy = parent.copy()
            parent_copy['alpha'] = parent['alpha'] / (len(children) + 1)
            new_gaussians.append(parent_copy)
            
            new_gaussians.extend(children)
            
    return new_gaussians

def merge_children(proposals, gamma_d, gamma_c):
    if not proposals:
        return []
    
    # Build adjacency graph
    n = len(proposals)
    adj = [[False] * n for _ in range(n)]
    for j in range(n):
        for k in range(j + 1, n):
            dj = proposals[j]
            dk = proposals[k]
            delta = dk['mu'] - dj['mu']
            
            # Symmetric Mahalanobis distance
            dist_sq = (delta.T @ dj['Sigma'].inverse() @ delta + 
                       delta.T @ dk['Sigma'].inverse() @ delta)
            
            # Color difference
            color_diff = torch.max(torch.abs(dj['c'] - dk['c']))
            
            if dist_sq <= gamma_d and color_diff <= gamma_c:
                adj[j][k] = True
                adj[k][j] = True
    
    # Connected components
    visited = [False] * n
    groups = []
    for j in range(n):
        if not visited[j]:
            group = []
            stack = [j]
            while stack:
                curr = stack.pop()
                if visited[curr]:
                    continue
                visited[curr] = True
                group.append(curr)
                for k in range(n):
                    if adj[curr][k] and not visited[k]:
                        stack.append(k)
            groups.append(group)
    
    merged = []
    for group in groups:
        M = len(group)
        mus = torch.stack([proposals[j]['mu'] for j in group])
        colors = torch.stack([proposals[j]['c'] for j in group])
        sigmas = torch.stack([proposals[j]['Sigma'] for j in group])
        
        mu_M = mus.mean(dim=0)
        c_M = colors.mean(dim=0)
        
        # Covariance merging
        Sigma_bar = sigmas.mean(dim=0)
        eigvals, eigvecs = torch.linalg.eigh(Sigma_bar)
        
        # Compute merged variance along each axis
        lambda_M = []
        for r in range(3):
            er = eigvecs[:, r]
            max_extent = 0
            for j in group:
                # 1-sigma extent of member j along er
                sigma_j = sigmas[j]
                # sigma_m,r is sqrt(er^T Sigma_m er) ? 
                # Source says: sigma_m,r is 1-sigma extent. 
                # Usually 1-sigma extent along axis e is sqrt(e^T Sigma e).
                # But formula uses max(|e^T (mu - mu_M)| + sigma_m,r)^2
                # Let's assume sigma_m,r is the standard deviation along that axis.
                sigma_m_r = torch.sqrt(er.T @ sigmas[j] @ er)
                dist = torch.abs(er.T @ (proposals[j]['mu'] - mu_M))
                extent = dist + sigma_m_r
                if extent > max_extent:
                    max_extent = extent
            lambda_M.append(max_extent**2)
        
        Sigma_M = eigvecs @ torch.diag(torch.tensor(lambda_M)) @ eigvecs.T
        
        merged.append({
            'mu': mu_M,
            'c': c_M,
            'Sigma': Sigma_M,
            'alpha': proposals[group[0]]['alpha'], # Inherited
            'R': compute_rotation_from_cov(Sigma_M) # Approximation or store R
        })
        
    return merged
```

## Operating characteristics
*   **Time Complexity:** The method adds overhead per densification step due to rendering $V$ views, computing error maps, and merging children. However, it reduces the total number of densification iterations required to achieve target quality, leading to net training time reductions of 9.2%–22.3% across benchmarks [S1, unofficial].
*   **Memory Complexity:** The number of Gaussians may increase slightly compared to shortened schedules but remains below full densification baselines in most cases [S1, unofficial]. The cross-view merging graph can become overly deep during low-resolution early training, increasing training time despite comparable Gaussian counts [S1, unofficial].
*   **Performance:** On MipNeRF360, 3DGS-AdpSplit achieves 27.54 PSNR, 0.810 SSIM, 0.225 LPIPS, 1391s training time, 2.55M Gaussians, and 105.13 FPS [S1, unofficial]. FastGS-AdpSplit matches full FastGS PSNR, improves LPIPS from 0.261 to 0.260, and trains in 143s (16.4% reduction) [S1, unofficial]. On Deep-Blending, FastGS-AdpSplit trains a scene in 83 seconds [S1, unofficial].
*   **Strengths:** Recovers lost quality from shortened training schedules; prioritizes small details; drop-in replacement for existing accelerators [S1, unofficial].
*   **Limitations:** Effective when densification is limited to 9k iterations; overly large early Gaussians can generate floating children from unreliable error regions; naive increase in random children amplifies noise [S1, unofficial].

## Background
AdpSplit was introduced in the preprint "AdpSplit: Error-Driven Adaptive Splitting for Faster Geometry Discovery in 3D Gaussian Splatting" (2026) [S1, unofficial]. It addresses the bottleneck in vanilla 3DGS where binary splitting requires many densification rounds to expose fine details, which is inefficient for accelerated training schedules [S1, unofficial]. It is integrated as a drop-in replacement for the split operator in pipelines such as 3DGS, MiniSplatting, DashGaussian, SpeedySplat, and FastGS [S1, unofficial].

## Sources
[S1] AdpSplit: Error-Driven Adaptive Splitting for Faster Geometry Discovery in 3D Gaussian Splatting, https://doi.org/10.48550/arXiv.2605.06876, fetched 2026-10-29 (unofficial)
