# Frosting Optimization and Parameterization

Also called: Gaussian Frosting Optimization

## Overview
This method defines the parameterization and optimization procedure for the 3D Gaussians within the Frosting layer, ensuring they remain confined to the irregular triangular prismatic cells derived from the underlying mesh. It solves the problem of uncontrolled Gaussian distribution in radiance fields by constraining positions to a volumetric layer while allowing flexible density allocation. It differs from unconstrained 3D Gaussian Splatting by enforcing geometric bounds via barycentric coordinates and by providing explicit user control over the total Gaussian budget, avoiding the stochastic densification/culling cycles of the parent method.

## How it works
**Inputs:**
- A set of $N$ prismatic cells $\mathcal{C}$, where each cell $c$ is defined by six vertices: three inner vertices $(\vec{v}_i + \delta^{in}_i \vec{n}_i)$ and three outer vertices $(\vec{v}_i + \delta^{out}_i \vec{n}_i)$ for $i \in \{0, 1, 2\}$, corresponding to the mesh face vertices $\vec{v}_i$ and normals $\vec{n}_i$ [S1].
- A target Gaussian budget $N$.
- An existing unconstrained 3D Gaussian representation (for color initialization) [S1].

**Outputs:**
- A set of $N$ Gaussians $\mathcal{G}$, each with a mean $\mu_g$, covariance $\Sigma_g$, opacity $\alpha_g$, and spherical harmonics coefficients for color [S1].

**Data Structures:**
Each Gaussian $g$ stores:
- Mean $\mu_g \in \mathbb{R}^3$: Parameterized by six barycentric coordinates.
- Covariance $\Sigma_g \in \mathbb{R}^{3 \times 3}$: Parameterized by a scaling vector $s_g \in \mathbb{R}^3$ and a quaternion $q_g \in \mathbb{R}^4$ [S1].
- Opacity $\alpha_g \in [0, 1]$.
- Color: Defined by spherical harmonics coordinates [S1].
- Barycentric parameters: Raw optimization variables for the six coordinates, which are mapped to valid barycentric weights via softmax.

**Parameterization Formula:**
The mean $\mu_g$ of a Gaussian inside a prismatic cell is computed as [S1]:
$$
\mu_g = \sum_{i=0}^{2} \left( b_g^{(i)} \left(\vec{v}_i + \delta^{out}_i \vec{n}_i\right) + \beta_g^{(i)} \left(\vec{v}_i + \delta^{in}_i \vec{n}_i\right) \right)
$$
where $b_g^{(i)}$ and $\beta_g^{(i)}$ are barycentric coordinates satisfying $\sum_{i=0}^{2} (b_g^{(i)} + \beta_g^{(i)}) = 1$. The parameters $\delta^{in}_i$ and $\delta^{out}_i$ are the inner and outer shifts derived from the unconstrained Gaussians [S1]. To ensure the sum-to-one constraint during optimization, the raw learnable parameters are passed through a softmax function [S1].

**Algorithm Steps:**

1.  **Cell Selection:** For each of the $N$ Gaussians to be instantiated:
    -   Select a prismatic cell $c \in \mathcal{C}$ with probability proportional to its volume.
    -   *Note:* To balance texture detail recovery in flat areas and volumetric rendering in fuzzy areas, half of the Gaussians ($N/2$) are selected with uniform probability across cells, and the other half ($N/2$) with probability proportional to cell volume [S1].

2.  **Barycentric Sampling:**
    -   Sample random barycentric coordinates $b_g^{(i)}$ and $\beta_g^{(i)}$ such that $\sum_{i=0}^{2} (b_g^{(i)} + \beta_g^{(i)}) = 1$ [S1].
    -   Compute $\mu_g$ using the formula above [S1].

3.  **Color Initialization:**
    -   Set the spherical harmonics coefficients of Gaussian $g$ to match the color of the closest Gaussian in the unconstrained representation [S1].

4.  **Other Parameter Initialization:**
    -   Reset opacity $\alpha_g$, scaling $s_g$, and rotation $q_g$ to fresh random values following the standard 3D Gaussian Splatting (3DGS) initialization strategy [S1]. This avoids local minima that the unconstrained Gaussians may have encountered, assuming the barycentric positions are a good starting point [S1].

5.  **Optimization:**
    -   Optimize the Gaussian parameters (barycentric coordinates, opacities, scales, rotations, and colors) using the standard photometric loss from Vanilla 3DGS: a combination of L1 loss and D-SSIM [S1].
    -   The number of Gaussians $N$ remains constant during this phase [S1].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def initialize_frosting_gaussians(prismatic_cells, budget_n, unconstrained_gaussians):
    """
    Initialize N Gaussians within the Frosting layer.
    
    Args:
        prismatic_cells: List of prismatic cell objects. Each cell has:
            - vertices: list of 6 vectors (3 inner, 3 outer)
            - volume: float
        budget_n: int, total number of Gaussians to create
        unconstrained_gaussians: List of Gaussian objects from the unconstrained representation
        
    Returns:
        list of Gaussian objects
    """
    gaussians = []
    
    # Split budget: half uniform, half volume-proportional
    n_uniform = budget_n // 2
    n_volume = budget_n - n_uniform
    
    # Precompute cumulative probabilities for volume-based selection
    total_volume = sum(c.volume for c in prismatic_cells)
    cumulative_probs = []
    acc = 0.0
    for c in prismatic_cells:
        acc += c.volume / total_volume
        cumulative_probs.append(acc)
        
    def select_cell_uniform():
        return random.choice(prismatic_cells)
        
    def select_cell_volume():
        r = random.random()
        for i, p in enumerate(cumulative_probs):
            if r <= p:
                return prismatic_cells[i]
        return prismatic_cells[-1]
        
    def sample_barycentric_coords():
        # Sample 6 values and normalize to sum to 1
        raw = [random.random() for _ in range(6)]
        total = sum(raw)
        return [x / total for x in raw]
        
    def compute_mean(coords, cell):
        # coords: [b0, b1, b2, beta0, beta1, beta2]
        b = coords[:3]
        beta = coords[3:]
        
        # cell.vertices: [v0+delta_out*n0, v1+delta_out*n1, v2+delta_out*n2,
        #                  v0+delta_in*n0,  v1+delta_in*n1,  v2+delta_in*n2]
        outer_verts = cell.vertices[0:3]
        inner_verts = cell.vertices[3:6]
        
        mean = [0.0, 0.0, 0.0]
        for i in range(3):
            # Outer contribution
            for k in range(3):
                mean[k] += b[i] * outer_verts[i][k]
            # Inner contribution
            for k in range(3):
                mean[k] += beta[i] * inner_verts[i][k]
        return mean

    def get_closest_color(pos, unconstrained_gs):
        # Find closest Gaussian in unconstrained set and return its color
        min_dist = float('inf')
        closest_color = None
        for g in unconstrained_gs:
            dist = sum((pos[k] - g.mean[k])**2 for k in range(3))
            if dist < min_dist:
                min_dist = dist
                closest_color = g.color # Assuming color is stored as RGB tuple/vector
        return closest_color

    def init_fresh_params():
        # Standard 3DGS initialization for opacity, scale, rotation
        opacity = random.uniform(0.1, 0.5) # [G] Typical range, exact values not specified in S1
        scale = [random.uniform(0.01, 0.1) for _ in range(3)] # [G]
        rotation = normalize_quaternion([random.gauss(0, 1) for _ in range(4)]) # [G]
        return opacity, scale, rotation

    for _ in range(n_uniform):
        cell = select_cell_uniform()
        coords = sample_barycentric_coords()
        mean = compute_mean(coords, cell)
        color = get_closest_color(mean, unconstrained_gaussians)
        opacity, scale, rotation = init_fresh_params()
        gaussians.append(Gaussian(mean=mean, color=color, opacity=opacity, scale=scale, rotation=rotation, barycentric_raw=coords))
        
    for _ in range(n_volume):
        cell = select_cell_volume()
        coords = sample_barycentric_coords()
        mean = compute_mean(coords, cell)
        color = get_closest_color(mean, unconstrained_gaussians)
        opacity, scale, rotation = init_fresh_params()
        gaussians.append(Gaussian(mean=mean, color=color, opacity=opacity, scale=scale, rotation=rotation, barycentric_raw=coords))
        
    return gaussians

def optimize_frosting(gaussians, images, cameras):
    """
    Optimize Frosting Gaussians using Vanilla 3DGS loss.
    """
    for epoch in range(max_epochs):
        for image, camera in zip(images, cameras):
            rendered_image = render_gaussians(gaussians, camera)
            loss = photometric_loss(rendered_image, image) # L1 + D-SSIM
            loss.backward()
            optimizer.step()
            optimizer.zero_grad()
```

**Edge Cases and Pitfalls:**
-   **Precision:** The barycentric coordinates must be strictly normalized. Using softmax on raw parameters ensures this numerically [S1].
-   **Local Minima:** Re-initializing opacities, scales, and rotations is critical. Using the unconstrained Gaussians' values for these parameters can lead to local minima because the unconstrained Gaussians may have converged to suboptimal configurations for the constrained layer [S1].
-   **Boundary Conditions:** The prismatic cells are defined by the mesh geometry. If the mesh is degenerate or has zero-volume cells, volume-proportional sampling may fail or produce undefined behavior. Not given in the sources.

## Operating characteristics
-   **Time Complexity:** Initialization is $O(N)$ where $N$ is the Gaussian budget. Optimization per step is comparable to Vanilla 3DGS, as the number of Gaussians is constant and no densification/culling is performed [S1].
-   **Memory Complexity:** $O(N)$, storing $N$ Gaussians with their parameters [S1].
-   **Performance:** The method allows precise control over the number of Gaussians, which can lead to more efficient memory usage compared to unconstrained 3DGS where the number of Gaussians can grow unpredictably [S1].
-   **Strengths:** Better control over Gaussian count; improved handling of fuzzy materials and complex geometry through volumetric layering; faster training due to stable Gaussian count [S1].
-   **Limitations:** Requires an underlying mesh to define prismatic cells; performance depends on the quality of the mesh and the shift parameters $\delta^{in/out}$ [S1].
-   **When to use:** When precise control over the number of Gaussians is required, or when dealing with fuzzy/complex geometry that benefits from volumetric representation within a bounded layer [S1].
-   **When not to use:** When no underlying mesh is available, or when the scene geometry is simple and does not require volumetric layering [S1].

## Background
The Frosting method was introduced by A. Guédon and V. Lepetit in 2024 [S1]. It extends 3D Gaussian Splatting by adding a "frosting" layer of Gaussians constrained to prismatic cells derived from a mesh, allowing for editable complex radiance fields with real-time rendering [S1]. The method builds on the optimization procedures of Vanilla 3DGS but introduces geometric constraints and explicit budget control [S1].

## Sources
[S1] Gaussian Frosting: Editable Complex Radiance Fields with Real-Time Rendering, https://doi.org/10.1007/978-3-031-73650-6_24, fetched 2026-10-26
