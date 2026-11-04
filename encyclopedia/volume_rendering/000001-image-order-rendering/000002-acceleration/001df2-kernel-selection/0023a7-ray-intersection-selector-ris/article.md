# Ray Intersection Selector (RIS)
Also called: IRIS (Intersection-aware Ray-based Implicit Editable Scenes)

## Overview
Ray Intersection Selector (RIS) is a sampling strategy for neural rendering that replaces stochastic volumetric sampling with analytical ray-primitive intersection detection to identify high-density regions for volumetric rendering [S1]. It solves the problem of empty space processing and costly 3D neighbor lookups by generating meaningful samples in a single pass, eliminating the need for auxiliary networks or voxel grids [S1]. Unlike [Kernel Selection](1647-kernel-selection.md) which selects a subset of Gaussoids based on density thresholds, RIS analytically determines precise intersection points between camera rays and 3D Gaussians, ensuring that every neural query contributes directly to surface reconstruction [S1].

## How it works
**Inputs and Outputs**
*   **Input:** A set of $N$ Neural Anchors (primitives) $\mathcal{P}$, a camera ray $r(t) = \mathbf{o} + t\mathbf{d}$, and a distance threshold $\tau_{\text{dist}}$.
*   **Output:** A sorted sequence of intersection points (samples) $\{x_k\}$ along the ray, and aggregated neural features for volumetric rendering.

**Data Structures**
The scene is parameterized as a set of $N$ Neural Anchors. Each anchor $i$ consists of:
1.  A 3D normal distribution defined by mean position $\boldsymbol{\mu}_i \in \mathbb{R}^3$ and covariance matrix $\boldsymbol{\Sigma}_i$.
2.  A logit parameter $\omega_i \in \mathbb{R}$ controlling opacity.
3.  A latent feature vector $\mathbf{f}_i \in \mathbb{R}^D$.
The covariance is factorized into a rotation matrix $R_i$ (stored as a quaternion) and a scaling vector $S_i$ [S1].

**Algorithm Steps**

1.  **Proxy Geometry Construction:** Bound each Gaussian primitive $i$ with an affine-transformed icosahedron to create a proxy geometry [S1].
2.  **Ray-Primitive Intersection:** Use the OptiX API to perform ray tracing against the proxy geometry, offloading intersection logic to dedicated RT cores [S1]. Solve the intersection equations for a ray traversing an oriented ellipsoid to determine precise interaction points [S1].
3.  **Sample Generation:** Generate a sorted sequence of intersection points (samples) $\{x_k\}$ only where density contribution is guaranteed to be significant, skipping empty space [S1].
4.  **Ray-Coherent Aggregation (RCA):** For each sample point $x_k$, define a neighborhood window $W_k$ of size $W = 2N + 1$ indices centered at $k$: $\{k-N, \dots, k, \dots, k+N\}$ [S1].
5.  **Validity Masking:** Apply a binary validity mask $M_{kj}$ to enforce spatial locality. A neighbor $j \in W_k$ is valid only if:
    *   It belongs to the same ray index.
    *   The Euclidean distance $\|\mathbf{x}_k - \boldsymbol{\mu}_j\|_2$ is below a fixed threshold $\tau_{\text{dist}}$ [S1].
6.  **Feature Aggregation:** For every valid neighbor $j$, compute the unnormalized logit $l_{kj}$ based on the Mahalanobis distance $\Delta^2(\mathbf{x}_k, \boldsymbol{\mu}_j, \boldsymbol{\Sigma}_j)$. Aggregate weights $w_{kj}$ are obtained by normalizing these logits (e.g., via softmax) [S1]. Synthesize a continuous latent vector $\mathbf{z}_k$ by weighting the latent features $\mathbf{f}_j$ of valid neighbors:
    $$ \mathbf{z}_k = \sum_{j \in W_k, M_{kj}=1} w_{kj} \mathbf{f}_j $$
7.  **Neural Decoding:** Use shallow Multi-Layer Perceptrons (MLPs) to decode the aggregated continuous latent vector $\mathbf{z}_k$ into view-dependent color and material density $\sigma$ [S1].
8.  **Volumetric Rendering:** The neural density $\sigma$ is spatially modulated by the explicit Gaussian opacity $\omega_i$ and integrated via point-based volumetric rendering to produce the final image [S1].

**Pseudocode**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import math

def ray_intersection_selector(anchors, ray_origin, ray_direction, tau_dist):
    """
    RIS: Analytically determines precise intersection points between camera rays and 3D Gaussians.
    
    Args:
        anchors: List of N Neural Anchors. Each anchor is a dict with keys:
                 'mu': (3,) numpy array, mean position.
                 'Sigma': (3,3) numpy array, covariance matrix.
                 'omega': float, logit opacity.
                 'f': (D,) numpy array, latent feature vector.
                 'proxy_icosahedron': Object supporting ray intersection test.
        ray_origin: (3,) numpy array, camera ray origin.
        ray_direction: (3,) numpy array, camera ray direction (normalized).
        tau_dist: float, Euclidean distance threshold for validity mask.
        
    Returns:
        samples: List of dicts, each containing:
                 'x': (3,) numpy array, intersection point.
                 'sigma': float, decoded density.
                 'color': (3,) numpy array, decoded color.
                 'opacity': float, modulated opacity.
    """
    # Step 1 & 2: Ray-Primitive Intersection using OptiX (simulated here)
    # In implementation, this calls OptiX API to trace against proxy icosahedrons.
    # Returns a list of intersection distances t sorted in ascending order.
    intersection_distances = trace_optix_rays(anchors, ray_origin, ray_direction)
    
    # Step 3: Generate sorted sequence of samples
    samples = []
    for t in intersection_distances:
        x = ray_origin + t * ray_direction
        
        # Step 4: Define neighborhood window in the sorted list
        # Note: In a full batch implementation, this is vectorized.
        # Here we assume we are processing one ray at a time for clarity,
        # but the window logic applies to the global sorted list of all ray intersections.
        # For a single ray, the window is simply the local neighbors in the list.
        # However, the source describes a global sorted list across all rays for RCA.
        # We assume 'anchors' and 'intersection_distances' are part of a larger batch context.
        # For this single-ray function, we approximate the window by looking at nearby anchors
        # based on distance along the ray, or assume the caller provides the window indices.
        
        # Simplified RCA for single ray context:
        # Find neighbors in the anchor list that are close to x along the ray
        # The source specifies a window of indices in the SORTED SEQUENCE of intersections.
        # Let's assume we have access to the global sorted list 'all_intersections'
        # and the index 'k' of the current intersection in that list.
        
        # Since this function is self-contained, we simulate the windowing by
        # selecting anchors whose centers are within a certain range along the ray.
        # This is a simplification of the strided tensor operation described.
        
        # Step 5 & 6: Ray-Coherent Aggregation
        valid_neighbors = []
        for i, anchor in enumerate(anchors):
            mu = anchor['mu']
            # Check validity mask
            dist = math.sqrt(np.sum((x - mu)**2))
            if dist < tau_dist:
                # Compute Mahalanobis distance
                diff = x - mu
                # Sigma_inv = np.linalg.inv(anchor['Sigma']) # Not given in sources, assumed available
                # mahal_dist_sq = diff @ Sigma_inv @ diff
                # For implementation completeness, we assume a standard Gaussian weight
                # based on the source's mention of Mahalanobis distance.
                # Since exact Sigma_inv usage isn't fully detailed in the snippet,
                # we use a placeholder for the weight calculation logic.
                weight = compute_mahalanobis_weight(x, mu, anchor['Sigma'])
                valid_neighbors.append((i, weight, anchor['f']))
        
        # Aggregate features
        if not valid_neighbors:
            # Fallback or empty feature if no neighbors
            z_k = np.zeros(anchors[0]['f'].shape) if anchors else np.array([])
        else:
            # Normalize weights
            total_weight = sum(w for _, w, _ in valid_neighbors)
            if total_weight == 0:
                z_k = np.zeros(anchors[0]['f'].shape) if anchors else np.array([])
            else:
                z_k = np.zeros(anchors[0]['f'].shape) if anchors else np.array([])
                for _, w, f in valid_neighbors:
                    z_k += (w / total_weight) * f
        
        # Step 7: Neural Decoding
        # Shallow MLPs decode z_k into color and density
        # Assuming a simple linear decode for pseudocode completeness
        sigma = decode_density(z_k)
        color = decode_color(z_k)
        
        # Step 8: Modulate by Gaussian opacity
        # The source says "neural density sigma is spatially modulated by the explicit Gaussian opacity omega_i"
        # Since we aggregated features, we might have an aggregated opacity or use the anchor's opacity.
        # The source implies the final integration uses this modulated density.
        # We'll assume the decoded sigma is the base, and we modulate it.
        # However, the source says "modulated by ... omega_i". 
        # In the aggregation step, we aggregated features f_i. 
        # We need to aggregate or select an opacity. 
        # Let's assume we also aggregate opacity or use the primary anchor's opacity.
        # For simplicity, we use the opacity of the closest valid neighbor or the anchor associated with the intersection.
        # The source doesn't specify how omega is aggregated, only that sigma is modulated by it.
        # We will assume the decoded sigma is already the final density for the sample point.
        
        samples.append({
            'x': x,
            'sigma': sigma,
            'color': color,
            'opacity': 1.0 # Placeholder for omega modulation logic
        })
        
    return samples

def compute_mahalanobis_weight(x, mu, Sigma):
    """
    Computes weight based on Mahalanobis distance.
    """
    diff = x - mu
    # Inverse of covariance matrix
    try:
        Sigma_inv = np.linalg.inv(Sigma)
    except np.linalg.LinAlgError:
        return 0.0
    
    mahal_dist_sq = diff @ Sigma_inv @ diff
    # Weight is typically exp(-0.5 * mahal_dist_sq)
    return math.exp(-0.5 * mahal_dist_sq)

def decode_density(z):
    """
    Placeholder for shallow MLP decoding density.
    """
    return max(0.0, np.dot(z, [1.0, 0.0, 0.0])) # Simplified

def decode_color(z):
    """
    Placeholder for shallow MLP decoding color.
    """
    return np.clip(np.dot(z, [0.0, 1.0, 0.0]), 0.0, 1.0) # Simplified
```

**Parameters**
*   $\tau_{\text{dist}}$: A fixed Euclidean distance threshold used in the validity mask to ensure spatial locality between samples. Value not given in the sources.

**Edge Cases and Pitfalls**
*   **Empty Space Processing:** Standard volumetric sampling queries empty space, wasting computation. RIS handles this by only evaluating structurally meaningful intersection points [S1].
*   **Geometric Discontinuity:** Naive index-based aggregation along the sorted ray list risks blending geometrically disjoint objects (e.g., foreground and background) that are adjacent in the list but distant in 3D space. This is mitigated by the validity mask $M_{kj}$ [S1].
*   **Precision:** The method relies on analytical intersection equations. Precision depends on the accuracy of the ray-triangle intersection within the OptiX API and the numerical stability of the covariance matrix inversion for Mahalanobis distance [S1].

## Operating characteristics
*   **Time Complexity:** RIS operates in a single pass, linearly scaling with the number of primitives and rays. It avoids the $O(N \log N)$ or $O(N^2)$ costs of KNN searches and the overhead of auxiliary network queries [S1].
*   **Memory Complexity:** Requires storage for $N$ Neural Anchors and their proxy geometries. No auxiliary voxel grids or occupancy grids are needed [S1].
*   **Measured Performance:** On the NeRF Synthetic dataset, IRIS achieves 6.038 FPS, approximately 30x higher than EKS (0.183 FPS) [S1]. Training time is reduced to 51 minutes compared to EKS's 26 hours [S1].
*   **Strengths:** Eliminates empty space processing; bypasses costly 3D KNN lookups; enables high-fidelity rendering with explicit geometry; supports topology-agnostic editing and physics simulation [S1].
*   **Limitations:** Requires GPU RT cores for optimal performance (OptiX); relies on the accuracy of the proxy geometry bounds [S1].
*   **When to Use:** When high rendering throughput is required and the scene is represented by explicit Gaussian primitives. Suitable for interactive editing and physics simulation [S1].
*   **When Not to Use:** When hardware lacks dedicated RT cores; when the scene representation is purely implicit without explicit primitives [S1].

## Background
IRIS (Intersection-aware Ray-based Implicit Editable Scenes) was introduced to address the computational inefficiencies of standard ray marching and stochastic sampling in neural rendering [S1]. It builds on the paradigms of Neural Radiance Fields (NeRF) and 3D Gaussian Splatting (3DGS), combining the view-dependent effects of NeRF with the explicit, editable nature of 3DGS [S1]. The method was developed to enable efficient and interactive scene editing while maintaining high visual fidelity [S1].

## Sources
[S1] IRIS: Intersection-aware Ray-based Implicit Editable Scenes, https://doi.org/10.48550/arXiv.2603.15368, fetched 2026-10-29
