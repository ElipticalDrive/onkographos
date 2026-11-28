# Surface-Aware Splat Graph Construction

Also called: Manifold-Aware Gaussian Graph Construction, Surface-Aligned Gaussian Connectivity

## Overview
Surface-Aware Splat Graph Construction establishes connectivity between 3D Gaussian primitives exclusively along the underlying surface manifold, filtering out spurious neighbors that are close in Euclidean space but distant along the surface [S1]. Unlike methods relying on Euclidean or Mahalanobis distance, which fail under challenging geometric conditions, this method defines adjacency as the intersection of Gaussian kernel occupancy regions, ensuring that graph edges represent true surface proximity [S1]. It serves as a prerequisite for Laplacian-based deformation and other surface-aware operations by providing a topologically correct connectivity graph for the Gaussian splats [S1].

## How it works
**Inputs:** A set of $N$ Gaussian kernels $\{G_i\}_{i=1}^N$ reconstructed from 2D images using a surface-aligned Gaussian Splatting method (specifically SpikingGS [S1]). Each kernel $G_i$ is defined by its position $p_i$, covariance matrix $\Sigma_i$, and opacity $\alpha_i$.
**Outputs:** A graph where nodes are the Gaussian kernels and edges exist between kernels $G_i$ and $G_j$ if their occupancy regions intersect within a tolerance.

**Data Structures:**
*   **Gaussian Kernel $G_i$:**
    *   $p_i \in \mathbb{R}^3$: Position.
    *   $\Sigma_i \in \mathbb{R}^{3 \times 3}$: Covariance matrix, factorized as $\Sigma_i = R_i S_i S_i^\top R_i^\top$.
    *   $R_i \in \mathbb{R}^{3 \times 3}$: Rotation matrix with columns $r_{i,1}, r_{i,2}, r_{i,3}$.
    *   $S_i \in \mathbb{R}^{3 \times 3}$: Scale matrix, diagonal with entries $\sigma_{i,1}, \sigma_{i,2}, 0$.
    *   $\alpha_i \in [0, 1]$: Opacity.
*   **Graph:** Adjacency list or matrix derived from intersection checks.

**Parameters:**
*   $c$: Minimum contribution threshold for rendered intensity. Typical value: $1/255$ [S1].
*   $\bar{V}_{ip}$: Activation cut-off threshold from the spiking neuron in SpikingGS. This is a learnable parameter that suppresses low-opacity parts [S1].
*   $\epsilon$: Tolerance for normal-wise offset to account for minor gaps between adjacent Gaussian kernels arising from joint optimization [S1].

**Steps:**

1.  **Define Occupancy Region:** For each kernel $G_i$, define the occupancy region $\Omega_i$ as the set of points $x \in \mathbb{R}^3$ where the kernel is considered active. Activity is determined by two constraints: the spiking activation threshold $\bar{V}_{ip}$ and the minimum intensity contribution $c$ [S1].
    The effective activation $\hat{G}_i(x)$ is:
    $$
    \hat{G}_i(x) = \begin{cases} 
    0 & \text{if } G_i(x) < \bar{V}_{ip} \\
    e^{-\frac{1}{2}(x-p_i)^\top \Sigma_i^{-1} (x-p_i)} & \text{otherwise}
    \end{cases}
    $$
    The occupancy region $\Omega_i$ is defined as:
    $$
    \Omega_i = \left\{ x \in \mathbb{R}^3 \mid \hat{G}_i(x) > \max\left(\bar{V}_{ip}, \frac{c}{\alpha_i}\right) \right\}
    $$
    [S1]

2.  **Parameterize Occupancy Region:** Since $\Sigma_i$ is rank-2 (due to the zero in $S_i$), $\Omega_i$ forms an ellipse in the local coordinate system. Using $S_i = \text{diag}(\sigma_{i,1}, \sigma_{i,2}, 0)$ and $R_i = [r_{i,1}, r_{i,2}, r_{i,3}]$, the region is parameterized as:
    $$
    \Omega_i = \left\{ p_i + \lambda_i \rho (\sigma_{i,1} \cos \theta \, r_{i,1} + \sigma_{i,2} \sin \theta \, r_{i,2}) \;\middle|\; \rho \in [0, 1), \theta \in (0, 2\pi] \right\}
    $$
    where the scaling factor $\lambda_i$ is:
    $$
    \lambda_i = \sqrt{-2 \max\left(\ln|\bar{V}_{ip}|, \ln\left(\frac{c}{\alpha_i}\right)\right)}
    $$
    [S1]

3.  **Compute Normal-Wise Offset:** To determine if two kernels $G_i$ and $G_j$ intersect, compute the normal-wise offset $\delta_{ij}$ from $G_i$ to $G_j$. This offset measures the minimum distance along the normal direction $r_{i,3}$ of $G_i$ required to reach the region $\Omega_j$:
    $$
    \delta_{ij} := \inf \{ |t| \mid \exists x \in \Omega_i : x + t r_{i,3} \in \Omega_j \}
    $$
    If no such $x$ exists, $\delta_{ij} = +\infty$ [S1].

4.  **Establish Connectivity:** Two Gaussians $G_i$ and $G_j$ are considered connected (intersecting) if $\delta_{ij} < \epsilon$. A zero offset indicates a perfect intersection, while values below $\epsilon$ account for minor gaps due to optimization artifacts [S1].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import numpy as np

def compute_lambda(alpha, c, v_bar_ip):
    """
    Compute the scaling factor lambda for a Gaussian kernel.
    
    Args:
        alpha (float): Opacity of the kernel.
        c (float): Minimum contribution threshold (e.g., 1/255).
        v_bar_ip (float): Spiking activation cut-off threshold.
        
    Returns:
        float: lambda_i
    """
    term1 = np.log(np.abs(v_bar_ip))
    term2 = np.log(c / alpha)
    max_val = max(term1, term2)
    # If max_val is positive, -2*max_val is negative, sqrt fails.
    # However, since G_i(x) < V_bar_ip leads to 0 activation,
    # the condition G_i(x) > V_bar_ip implies the exponent is negative.
    # The derivation assumes valid active regions where the exponent is negative.
    # In practice, ensure arguments to log are valid and max_val < 0 for active regions.
    if max_val >= 0:
        return 0.0
    return np.sqrt(-2 * max_val)

def get_ellipse_points(p_i, R_i, S_i, lambda_i, num_theta=100):
    """
    Generate points on the boundary of the occupancy region ellipse.
    
    Args:
        p_i (np.array): Position (3,).
        R_i (np.array): Rotation matrix (3,3).
        S_i (np.array): Scale matrix diagonal values (sigma1, sigma2, 0).
        lambda_i (float): Scaling factor.
        num_theta (int): Number of samples for theta.
        
    Returns:
        np.array: Points on the ellipse boundary (num_theta, 3).
    """
    sigma1, sigma2, _ = S_i
    r1, r2, _ = R_i.T # Columns of R_i
    
    thetas = np.linspace(0, 2 * np.pi, num_theta, endpoint=False)
    rho = 1.0 # Boundary
    
    # Ellipse in local space: lambda * (sigma1 * cos(theta) * r1 + sigma2 * sin(theta) * r2)
    # Note: The formula uses rho in [0,1). For boundary check, rho=1 is sufficient.
    points_local = lambda_i * (sigma1 * np.cos(thetas) * r1 + sigma2 * np.sin(thetas) * r2)
    
    # Transform to world space
    points_world = p_i + points_local
    return points_world

def check_intersection(G_i, G_j, epsilon):
    """
    Check if Gaussian G_i and G_j intersect using normal-wise offset.
    
    Args:
        G_i (dict): Kernel i with keys 'p', 'R', 'S', 'alpha', 'v_bar_ip'.
        G_j (dict): Kernel j with keys 'p', 'R', 'S', 'alpha', 'v_bar_ip'.
        epsilon (float): Tolerance for offset.
        
    Returns:
        bool: True if intersecting, False otherwise.
    """
    # 1. Compute lambda for G_i
    lambda_i = compute_lambda(G_i['alpha'], 1/255, G_i['v_bar_ip'])
    
    # 2. Get boundary points of Omega_i
    # We approximate the infimum by checking if any point on Omega_i's boundary
    # is close to Omega_j. However, the definition is inf |t| such that x + t*r_i3 is in Omega_j.
    # A robust implementation requires solving for t.
    # Simplified approach for implementation:
    # Project the center of G_j onto the plane of G_i? No, G_i is an ellipse.
    # The definition: inf |t| | exists x in Omega_i : x + t * r_i3 in Omega_j.
    
    # Let's approximate by sampling the boundary of Omega_i and checking distance to Omega_j along r_i3.
    # This is an approximation of the infimum.
    
    p_i = G_i['p']
    r_i3 = G_i['R'][:, 2] # Third column of rotation matrix
    S_i = np.diag(G_i['S']) # Assuming S is stored as vector or diag matrix
    
    # Generate boundary points of Omega_i
    # Note: S_i in formula is diag(sigma1, sigma2, 0). 
    # We need the actual sigma values.
    sigma1, sigma2, _ = G_i['S'] # Assuming G_i['S'] is [sigma1, sigma2, 0] or similar
    
    # Re-calculate lambda if not precomputed
    # lambda_i = compute_lambda(...)
    
    # Sample points on the ellipse boundary of Omega_i
    num_samples = 100
    thetas = np.linspace(0, 2 * np.pi, num_samples, endpoint=False)
    
    r1 = G_i['R'][:, 0]
    r2 = G_i['R'][:, 1]
    
    # Points on the boundary of Omega_i
    # x = p_i + lambda_i * (sigma1 * cos(theta) * r1 + sigma2 * sin(theta) * r2)
    cos_t = np.cos(thetas)
    sin_t = np.sin(thetas)
    
    x_boundary = p_i + lambda_i * (sigma1 * cos_t[:, None] * r1 + sigma2 * sin_t[:, None] * r2)
    
    # For each point x on Omega_i boundary, find t such that x + t * r_i3 is in Omega_j
    # Omega_j is also an ellipse.
    # This is a complex geometric intersection.
    # The source defines delta_ij as inf |t| | exists x in Omega_i : x + t * r_i3 in Omega_j.
    
    # Approximation:
    # 1. Project x_boundary onto the normal plane of r_i3? No.
    # 2. Check if the line segment from x in direction r_i3 intersects Omega_j.
    
    # Since Omega_j is an ellipse in 3D, checking intersection of a ray with an ellipse is non-trivial.
    # However, for the purpose of this article, we state the definition.
    # Implementation detail [G]: A practical approximation is to check if the distance 
    # from the center of G_j to the plane of G_i (defined by p_i and normal r_i3) 
    # is small, and if the projected point lies within the ellipse of G_j.
    # But the source specifies the infimum over x in Omega_i.
    
    # Let's implement a simplified check based on the definition:
    # We want to find min |t| such that x + t*r_i3 in Omega_j for some x in Omega_i.
    # This is equivalent to finding the minimum distance along r_i3 between the two ellipses.
    
    # For a robust implementation, one would solve the quadratic equation for ray-ellipse intersection
    # for each sample point on Omega_i and take the minimum |t|.
    
    min_t = np.inf
    
    # Define function to check if point y is in Omega_j
    def is_in_omega_j(y, G_j):
        # Transform y to local space of G_j
        p_j = G_j['p']
        R_j = G_j['R']
        S_j = G_j['S'] # [sigma1, sigma2, 0]
        alpha_j = G_j['alpha']
        v_bar_j = G_j['v_bar_ip']
        
        # Local coordinates
        diff = y - p_j
        local_diff = R_j.T @ diff
        
        # Check against ellipse equation: (x'/sigma1)^2 + (y'/sigma2)^2 <= 1
        # And activation thresholds
        sigma1_j, sigma2_j, _ = S_j
        
        # Lambda for G_j
        lambda_j = compute_lambda(alpha_j, 1/255, v_bar_j)
        
        # The point y is in Omega_j if it is within the scaled ellipse
        # The ellipse equation in local space is (u/sigma1)^2 + (v/sigma2)^2 <= 1
        # But Omega_j is scaled by lambda_j.
        # So: (u / (lambda_j * sigma1_j))^2 + (v / (lambda_j * sigma2_j))^2 <= 1
        
        u, v, w = local_diff
        denom1 = lambda_j * sigma1_j
        denom2 = lambda_j * sigma2_j
        
        if denom1 == 0 or denom2 == 0:
            return False
            
        dist_sq = (u / denom1)**2 + (v / denom2)**2
        return dist_sq <= 1.0

    # Check each sample point on Omega_i boundary
    for k in range(num_samples):
        x = x_boundary[k]
        
        # We need to find t such that x + t * r_i3 is in Omega_j.
        # Ray: r(t) = x + t * r_i3
        # We want to find t that minimizes |t| such that r(t) in Omega_j.
        
        # This is a ray-ellipse intersection problem.
        # Let's approximate by checking a range of t values or solving analytically.
        # Analytical solution for ray-ellipse intersection:
        # Ellipse in local space of G_j: (u/s1)^2 + (v/s2)^2 <= 1, w=0 (effectively, since rank 2)
        # But G_j is in 3D. The ellipse lies in the plane spanned by r_j1 and r_j2.
        # The ray intersects the plane of the ellipse at some t_plane.
        # Then check if the intersection point is inside the ellipse.
        
        # Plane of G_j: normal n_j = r_j3. Point p_j.
        # Ray: x + t * r_i3.
        # Intersection with plane: (x + t * r_i3 - p_j) . n_j = 0
        # t * (r_i3 . n_j) = (p_j - x) . n_j
        # t = (p_j - x) . n_j / (r_i3 . n_j)
        
        n_j = G_j['R'][:, 2]
        denom = np.dot(r_i3, n_j)
        
        if np.abs(denom) > 1e-6:
            t_plane = np.dot(G_j['p'] - x, n_j) / denom
            intersection_point = x + t_plane * r_i3
            
            if is_in_omega_j(intersection_point, G_j):
                min_t = min(min_t, np.abs(t_plane))
        else:
            # Ray is parallel to the plane of G_j
            # Check if the ray lies in the plane
            if np.abs(np.dot(x - G_j['p'], n_j)) < 1e-6:
                # Ray is in the plane. Check if it intersects the ellipse.
                # This is a 2D ray-ellipse intersection.
                # Transform ray to local space of G_j.
                # This is complex. For the purpose of the article, we note this edge case.
                pass

    return min_t < epsilon

def build_surface_aware_graph(gaussians, epsilon):
    """
    Build the surface-aware splat graph.
    
    Args:
        gaussians (list of dict): List of Gaussian kernels.
        epsilon (float): Tolerance.
        
    Returns:
        list of list: Adjacency list.
    """
    N = len(gaussians)
    adj = [[] for _ in range(N)]
    
    for i in range(N):
        for j in range(i + 1, N):
            if check_intersection(gaussians[i], gaussians[j], epsilon):
                adj[i].append(j)
                adj[j].append(i)
                
    return adj
```

## Operating characteristics
*   **Time Complexity:** The graph construction requires checking intersections for all pairs of Gaussians, resulting in $O(N^2)$ complexity, where $N$ is the number of Gaussian kernels [S1].
*   **Memory Complexity:** $O(N^2)$ in the worst case for a dense adjacency matrix, or $O(E)$ for an adjacency list, where $E$ is the number of edges [S1].
*   **Strengths:** It accurately captures surface topology by ignoring Euclidean proximity that does not correspond to surface connectivity, which is crucial for deformation tasks [S1].
*   **Limitations:** The $O(N^2)$ pairwise check can be computationally expensive for large scenes. It relies on the quality of the initial surface-aligned reconstruction (e.g., SpikingGS) [S1].
*   **When to use:** When performing surface-aware operations like Laplacian-based deformation on Gaussian splats [S1].
*   **When not to use:** For standard rendering where only visibility and color are needed, as standard rasterization does not require this graph structure [S1].

## Background
Surface-Aware Splat Graph Construction was introduced in the paper "Proxy-Free Gaussian Splats Deformation with Splat-Based Surface Estimation" [S1]. The method addresses the limitation of existing point-based distance metrics (Euclidean, Mahalanobis) in establishing connectivity along the surface manifold, particularly in challenging geometric conditions [S1]. It builds upon surface-aligned Gaussian Splatting methods, specifically SpikingGS, to ensure that the Gaussian primitives are aligned with the underlying surface geometry [S1]. The method is used to construct a Laplacian operator for proxy-free deformation of the Gaussian splats [S1].

## Sources
[S1] Proxy-Free Gaussian Splats Deformation with Splat-Based Surface Estimation, https://doi.org/10.1109/3DV69130.2026.00155, fetched 2026-10-30
