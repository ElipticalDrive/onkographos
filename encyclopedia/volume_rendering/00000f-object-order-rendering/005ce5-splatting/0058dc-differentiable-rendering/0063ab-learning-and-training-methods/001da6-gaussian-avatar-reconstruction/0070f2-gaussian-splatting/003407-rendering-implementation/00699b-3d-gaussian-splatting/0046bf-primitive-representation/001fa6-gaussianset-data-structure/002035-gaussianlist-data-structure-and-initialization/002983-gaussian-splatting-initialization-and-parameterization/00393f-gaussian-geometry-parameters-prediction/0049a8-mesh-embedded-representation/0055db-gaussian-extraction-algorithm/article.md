# Gaussian Extraction Algorithm
Also called: Discretization Algorithm, Octree-Guided Geometry Sampling, Optimizing Geometry with GauPF, Octree-guided Sampling

## Overview
The Gaussian Extraction Algorithm extracts discrete 3D Gaussian primitives from continuous Gaussian Splatting Functions (GauPF) at arbitrary densities [S1]. It solves the problem of converting a continuous, neural-network-parameterized probability field into a finite set of splattable Gaussians, analogous to how Marching Cubes extracts meshes from Signed Distance Functions [S1]. Unlike sibling methods that rely on fixed initialization or mesh embedding constraints, this algorithm uses an octree-guided progressive sampling strategy to locate high-probability regions, followed by gradient-based optimization to refine those locations [S1].

## How it works
**Inputs and Outputs:**
*   **Input:** A trained Gaussian Probability Function $\psi_{pf}$ (neural network), a Gaussian Color Function $\psi_{cf}$, a Gaussian Transform Function $\psi_{tf}$, a triplane representation $t$, and a target number of Gaussians $N$ [S1].
*   **Output:** A set of $N$ discrete 3D Gaussians $\hat{G} = \{\hat{\sigma}_i, \hat{c}_i, \hat{r}_i, \hat{s}_i, \hat{o}_i\}_{i=1}^N$, where $\hat{\sigma}_i$ is the center, $\hat{c}_i$ is the color, and $\{\hat{r}_i, \hat{s}_i, \hat{o}_i\}$ are rotation, scale, and opacity [S1].

**Data Structures:**
*   **Octree:** A hierarchical 3D grid used to progressively subdivide space. The depth is limited to $L$ layers [S1].
*   **Proxy Points:** A set of learnable 3D coordinates $\{\rho_i\}_{i=1}^N$ representing coarse initial locations of Gaussian centers [S1].
*   **Triplane $t$:** A set of feature planes used for attribute interpolation [S1].

**Steps:**
1.  **Octree Construction:** Build an octree progressively over $L$ layers [S1]. At each depth, identify 3D regions with large probabilities using $\psi_{pf}$ and subdivide only those regions [S1].
2.  **Coarse Sampling:** After $L$ layers of subdivision, uniformly sample $N$ 3D points within the leaf nodes of the octree [S1]. These points form the initial proxy points $\{\rho_i\}_{i=1}^N$ [S1].
3.  **Geometry Optimization:** Optimize the positions of the proxy points $\rho_i$ to maximize the probability assigned by $\psi_{pf}$ [S1]. The positions $\rho_i$ are treated as learnable parameters [S1].
4.  **Attribute Extraction:** Once the optimized centers $\hat{\sigma}_i$ are found, extract color and transform attributes using the triplane $t$ and the respective predictor functions [S1].

**Formulas:**
*   **Geometry Loss:** The proxy points are optimized to maximize the Gaussian probability [S1]. The loss function is defined as [S1]:
    $$
    L_{Geo} = -\frac{1}{N} \sum_{i=1}^N \psi_{pf}(\rho_i)
    $$
    where $\rho_i$ are the learnable positions of the proxy points [S1].
*   **Color Extraction:** The color $\hat{c}_i$ for the $i$-th Gaussian is extracted by interpolating the triplane $t$ at the optimized center $\hat{\sigma}_i$ and passing it through the color predictor $\psi_{cf}$ [S1]:
    $$
    \hat{c}_i = \psi_{cf}(\text{interp}(t, \hat{\sigma}_i))
    $$
*   **Transform Extraction:** The rotation $\hat{r}_i$, scale $\hat{s}_i$, and opacity $\hat{o}_i$ are extracted similarly [S1]:
    $$
    \{\hat{r}_i, \hat{s}_i, \hat{o}_i\} = \psi_{tf}(\text{interp}(t, \hat{\sigma}_i))
    $$

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import numpy as np

def gaussian_extraction_algorithm(pf_net, cf_net, tf_net, triplane, N, L, bounds, lr, max_iter):
    """
    Extracts N discrete Gaussians from continuous Gaussian Splatting Functions.
    
    Args:
        pf_net: Gaussian Probability Function network (psi_pf)
        cf_net: Gaussian Color Function network (psi_cf)
        tf_net: Gaussian Transform Function network (psi_tf)
        triplane: Feature planes t for attribute interpolation
        N: Number of target Gaussians
        L: Number of octree subdivision layers
        bounds: 3D bounding box [min_x, min_y, min_z, max_x, max_y, max_z]
        lr: Learning rate for optimization
        max_iter: Number of optimization iterations
        
    Returns:
        gaussians: List of N Gaussian dictionaries with sigma, c, r, s, o
    """
    # Step 1 & 2: Octree-guided coarse sampling
    # Build octree over L layers, subdividing regions where pf_net probability is high [S1]
    octree = _build_octree(pf_net, bounds, L)
    # Uniformly sample N points from leaf nodes as initial proxy points rho_i [S1]
    proxy_points = _sample_octree_leaves(octree, N)

    # Step 3: Geometry Optimization
    # Proxy points are learnable parameters [S1]
    proxy_points.requires_grad = True
    optimizer = _Adam([proxy_points], lr=lr)

    for _ in range(max_iter):
        probabilities = pf_net(proxy_points)
        loss = -np.mean(probabilities)
        optimizer.zero_grad()
        loss.backward()
        optimizer.step()

    sigma_hat = proxy_points.detach()

    # Step 4: Attribute Extraction
    gaussians = []
    for i in range(N):
        center = sigma_hat[i]
        features = _interpolate_triplane(triplane, center)
        c_hat = cf_net(features)
        r_hat, s_hat, o_hat = tf_net(features)
        gaussians.append({'sigma': center, 'c': c_hat, 'r': r_hat, 's': s_hat, 'o': o_hat})
    return gaussians

def _build_octree(pf_net, bounds, max_depth):
    """Recursively subdivide bounding box where pf_net probability exceeds threshold [G]."""
    def _subdivide(node_bounds, depth):
        if depth >= max_depth:
            return [node_bounds]
        mid = (node_bounds[:3] + node_bounds[3:]) / 2.0
        children = []
        for i in range(2):
            for j in range(2):
                for k in range(2):
                    child_min = np.array([node_bounds[i*2] if i==0 else mid[i*2],
                                          node_bounds[j*2+1] if j==0 else mid[j*2+1],
                                          node_bounds[k*2+2] if k==0 else mid[k*2+2]])
                    child_max = np.array([mid[0] if i==0 else node_bounds[3],
                                          mid[1] if j==0 else node_bounds[4],
                                          mid[2] if k==0 else node_bounds[5]])
                    child_bounds = np.concatenate([child_min, child_max])
                    center = (child_min + child_max) / 2.0
                    if pf_net(center.reshape(1, -3)) > 0.5:
                        children.extend(_subdivide(child_bounds, depth + 1))
        return children if children else [node_bounds]
    return _subdivide(bounds, 0)

def _sample_octree_leaves(octree, N):
    """Uniformly sample N points from leaf node bounds [G]."""
    points = []
    for bounds in octree:
        points.append(np.random.uniform(bounds[:3], bounds[3:]))
    points = np.array(points)
    if len(points) < N:
        points = np.vstack([points] * (N // len(points) + 1))
    return points[:N]

def _Adam(params, lr):
    """Minimal Adam optimizer implementation [G]."""
    state = {'m': [np.zeros_like(p) for p in params], 'v': [np.zeros_like(p) for p in params], 't': 0}
    def zero_grad():
        for p in params:
            p.grad = np.zeros_like(p)
    def step():
        state['t'] += 1
        beta1, beta2, eps = 0.9, 0.999, 1e-8
        for i, p in enumerate(params):
            state['m'][i] = beta1 * state['m'][i] + (1 - beta1) * p.grad
            state['v'][i] = beta2 * state['v'][i] + (1 - beta2) * p.grad**2
            m_hat = state['m'][i] / (1 - beta1**state['t'])
            v_hat = state['v'][i] / (1 - beta2**state['t'])
            p -= lr * m_hat / (np.sqrt(v_hat) + eps)
    return type('Adam', (), {'zero_grad': zero_grad, 'step': step})

def _interpolate_triplane(triplane, center):
    """Trilinear interpolation of triplane features at center [G]."""
    features = []
    for plane in triplane:
        x, y, z = center
        u, v = np.clip((x - plane.min_x) / plane.width, 0, 1), np.clip((y - plane.min_y) / plane.height, 0, 1)
        features.append(plane.data[int(v*plane.H), int(u*plane.W)])
    return np.concatenate(features)
```

**Parameters:**
*   $N$: The number of Gaussians to extract. The algorithm supports arbitrary numbers, enabling scalable generation [S1].
*   $L$: The number of octree subdivision layers. Not given in the sources.
*   Learning Rate: Not given in the sources [G].
*   Max Iterations: Not given in the sources [G].

**Edge Cases and Pitfalls:**
*   **Computational Cost:** Naive uniform sampling across the entire 3D space is computationally expensive and fails to accurately locate high-probability regions [S1]. The octree-guided approach mitigates this by focusing computation on relevant regions [S1].
*   **Arbitrary Density:** The method allows generating Gaussians at any density $N$, unlike fixed-grid methods [S1].
*   **Precision:** Gradient optimization of proxy points may converge to local maxima in the probability field; multiple restarts or adaptive thresholds are recommended [G].

## Operating characteristics
*   **Time Complexity:** Not given in the sources [G]. The complexity depends on the octree depth $L$, the number of subdivisions, and the optimization iterations [G].
*   **Memory Complexity:** Not given in the sources [G]. Depends on the octree structure and the number of proxy points $N$ [G].
*   **Measured Performance:** Evaluated on ShapeNet airplane and chair classes using FID and KID metrics [S1].
*   **Strengths:**
    *   **Scalability:** Can generate Gaussian primitives at arbitrary numbers [S1].
    *   **Quality:** Generates native 3DGS without voxelization, preserving unimpaired quality [S1].
    *   **Generality:** Applicable to unconditional generation, text/image conditional generation, and point-to-gaussian generation [S1].
*   **Limitations:**
    *   Requires a trained continuous Gaussian Splatting Function (GauPF) as input [S1].
    *   Optimization of proxy points requires iterative gradient descent, which adds time to the extraction process compared to direct sampling [S1].
*   **When to use:** When you need to convert a continuous, learned Gaussian Splatting Function into a discrete set of splats for rendering or further processing, especially when the number of splats needs to be flexible [S1].
*   **When not to use:** When real-time extraction is required without optimization overhead, or when the input is already discrete Gaussians [G].

## Background
The Gaussian Extraction Algorithm was introduced in the DiffGS (Functional Gaussian Splatting Diffusion) paper [S1]. It was designed to complement the generation of continuous Gaussian Splatting Functions by a latent diffusion model, providing a way to discretize these functions into usable 3D Gaussian Splatting primitives [S1]. The approach is inspired by the Marching Cubes algorithm, which extracts meshes from Signed Distance Functions [S1].

## Sources
[S1] DiffGS: Functional Gaussian Splatting Diffusion, https://arxiv.org/abs/2410.19657v2, fetched 2026-10-26
