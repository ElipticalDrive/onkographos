# Objects as Volumes (OaV) Attenuation-Vacancy Relation

Also called: OaV Relation, Stochastic Surface Occupancy Model

## Overview
The Objects as Volumes (OaV) Attenuation-Vacancy Relation provides a theoretical framework that interprets a scene represented by semi-transparent volumetric primitives (such as 3D Gaussians) as a stochastic surface. It links the geometric probability of a point being unoccupied (vacancy) to the optical attenuation coefficient used in standard volumetric rendering. This method enables the extraction of explicit, sharp geometric surfaces from continuous, semi-transparent volume representations by establishing a reciprocal exponential transport relationship between the two domains.

## How it works
The method relies on the assumption that light transport follows an exponential decay model and that the attenuation is reciprocal. It computes a vacancy field from the rendering parameters and uses it to define a surface via an isosurface extraction (e.g., Marching Cubes).

**Inputs:**
- Set of $N$ Gaussian primitives, each defined by opacity $\alpha_i \in [0, 1]$, mean $\mu_i \in \mathbb{R}^3$, and precision matrix $\Sigma_i^{-1} \in \mathbb{S}_{++}^3$ [S1].
- Set of training camera rays $\mathcal{T}_c$, where each ray is defined by origin $\mathbf{o}$ and direction $\mathbf{w} \in \mathcal{S}^2$ [S1].

**Outputs:**
- A vacancy field $v: \mathbb{R}^3 \rightarrow [0, 1]$, representing the probability that a point is outside the opaque volume [S1].
- An occupancy field $O(\mathbf{x}) = 1 - v(\mathbf{x})$ [S1].

**Data Structures:**
- **Gaussian Primitive:** Stores $\alpha_i, \mu_i, \Sigma_i^{-1}$.
- **Ray:** Stores $\mathbf{o}, \mathbf{w}$.
- **Vacancy Grid:** A 3D grid storing $v(\mathbf{x})$ values for mesh extraction [S1].

**Core Formula:**
The fundamental relation connects the vacancy $v(\mathbf{x})$ to the attenuation $\sigma(\mathbf{x}, \mathbf{w})$:

$$ \sigma(\mathbf{x}, \mathbf{w}) = |\mathbf{w} \cdot \nabla \log v(\mathbf{x})| $$

This holds under the reciprocity assumption:
$$ \sigma(\mathbf{x}, \mathbf{w}) = \sigma(\mathbf{x}, -\mathbf{w}) $$

**Vacancy Estimation via Transmittance Lower Bound:**
Direct integration of the vector field $\nabla \log v$ is sensitive to noise from non-oriented or hidden Gaussians. Instead, a robust lower bound is computed using training camera transmittance [S1].

1. For a query point $\mathbf{x} \in \mathbb{R}^3$, iterate over all training camera rays $(\mathbf{o}, \mathbf{w}) \in \mathcal{T}_c$.
2. For each ray, determine if it passes through $\mathbf{x}$. If $\mathbf{x} = \mathbf{o} + t\mathbf{w}$ for some $t > 0$, compute the transmittance $T_{\mathbf{o},\mathbf{w}}(t)$ along the ray segment from $\mathbf{o}$ to $\mathbf{x}$.
3. The transmittance is calculated using the non-oriented Gaussian formulation for robustness against outlier Gaussians [S1]:
   $$ T_{\mathbf{o},\mathbf{w}}(t) = \prod_{i=1}^{N} (1 - \bar{G}_{\mathbf{o},\mathbf{w}}^{(i)}(t)) $$
   where $\bar{G}_{\mathbf{o},\mathbf{w}}^{(i)}(t)$ is the contribution of the $i$-th Gaussian at the intersection point, clipped to the ray segment [S1].
4. The vacancy estimate $v(\mathbf{x})$ is the maximum transmittance observed across all training rays passing through $\mathbf{x}$:
   $$ v(\mathbf{x}) \geq \max_{(\mathbf{o},\mathbf{w}) \in \mathcal{T}_c} \{ T_{\mathbf{o},\mathbf{w}}(t) : \mathbf{x} = \mathbf{o} + t\mathbf{w}, t > 0 \} $$

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def compute_vacancy_lower_bound(x, gaussians, training_cameras):
    """
    Computes the lower bound of vacancy v(x) for a point x.
    
    Args:
        x: Query point in 3D space (numpy array, shape (3,)).
        gaussians: List of Gaussian primitives. Each element is a dict/object 
                   with attributes: alpha, mean, precision_matrix.
        training_cameras: List of training camera rays. Each element is a tuple 
                          (origin, direction).
    
    Returns:
        v_est: Estimated vacancy value in [0, 1].
    """
    max_transmittance = 0.0
    
    for origin, direction in training_cameras:
        # Check if ray passes through x
        # Solve x = origin + t * direction for t > 0
        # This requires solving a linear system or checking alignment.
        # In practice, we check if x lies on the ray segment.
        # For robustness, we assume we only query points known to be near rays,
        # or we project x onto the ray.
        
        # Vector from origin to x
        vec = x - origin
        
        # Project vec onto direction to find t
        # t = (vec . direction) / (direction . direction)
        # Assuming direction is normalized, denominator is 1.
        t = np.dot(vec, direction)
        
        # Check if t is positive and x is approximately on the ray
        if t <= 0:
            continue
            
        # Check distance from ray to x (should be near zero)
        # projection_point = origin + t * direction
        # dist = np.linalg.norm(x - projection_point)
        # if dist > threshold: continue 
        # [G] Implementation detail: In a grid-based approach, one typically 
        # iterates over rays that are known to intersect the voxel containing x.
        
        # Compute transmittance from origin to x along this ray
        transmittance = 1.0
        
        for gaussian in gaussians:
            # Calculate contribution of this Gaussian at point p = origin + t * direction
            # Using non-oriented formulation for robustness [S1]
            p = origin + t * direction
            
            # Gaussian value at p
            # G_i(p) = alpha_i * exp(-0.5 * (p - mu_i)^T * Sigma_inv_i * (p - mu_i))
            diff = p - gaussian.mean
            exponent = -0.5 * np.dot(diff, np.dot(gaussian.precision_matrix, diff))
            g_val = gaussian.alpha * np.exp(exponent)
            
            # Clip g_val to [0, 1] to ensure valid probability
            g_val = np.clip(g_val, 0.0, 1.0)
            
            transmittance *= (1.0 - g_val)
            
            # Early exit if transmittance is negligible
            if transmittance < 1e-6:
                break
        
        # Update maximum transmittance
        if transmittance > max_transmittance:
            max_transmittance = transmittance
            
    return max_transmittance
```

**Edge Cases and Pitfalls:**
- **Non-oriented Gaussians:** Direct integration of $\nabla \log v$ fails if Gaussians do not follow the oriented assumption, leading to noisy results [S1]. The transmittance lower bound is robust to this [S1].
- **Hidden/Floaters:** Gaussians trapped inside the volume or floaters can perturb direct integration. The lower bound method is robust because transmittance is non-decreasing and depends mainly on visible Gaussians [S1].
- **Holes:** Remaining holes in the Gaussian coverage can affect direct integration but are handled robustly by the max-transmittance approach [S1].

## Subtopics

### Vacancy Estimation via Transmittance Lower Bound
A specific implementation technique for computing the vacancy field $v(\mathbf{x})$. Instead of integrating the gradient of the log-vacancy, it computes a lower bound by taking the maximum transmittance over all training camera rays passing through $\mathbf{x}$. This method is chosen for its robustness to outlier Gaussians, floaters, and holes in the Gaussian coverage, as it relies on the non-decreasing nature of transmittance and primarily considers visible Gaussians [S1].

## Operating characteristics
- **Time Complexity:** Computing the vacancy for a single point requires iterating over all training cameras and all Gaussians for each ray passing through the point. This is $O(|\mathcal{T}_c| \cdot N)$ per point, which is expensive for dense grids. [G] In practice, spatial indexing (e.g., voxel grids) is used to only query relevant rays and Gaussians.
- **Memory Complexity:** Requires storing the Gaussian primitives and the training camera rays. The output vacancy field scales with the resolution of the grid used for mesh extraction.
- **Strengths:** Provides a principled link between volumetric rendering and surface geometry. The lower bound estimation is robust to common artifacts in Gaussian Splatting (floaters, unoriented Gaussians) [S1].
- **Limitations:** The method assumes reciprocity of attenuation, which may not hold for all media. The lower bound is an approximation and may underestimate the true vacancy in regions with sparse camera coverage [S1].

## Background
The Objects as Volumes (OaV) framework was introduced to bridge the gap between volumetric rendering (used in NeRF and 3DGS) and explicit surface representations. It interprets the scene as a stochastic surface, where occupancy is the probability of a point being occupied. This framework grounds the theoretical justification of Neural SDFs and links the vacancy to the attenuation of standard volumetric ray-marching [S1]. The specific application to 3D Gaussian Splatting for surface reconstruction was detailed in "From Blobs to Spokes: High-Fidelity Surface Reconstruction via Oriented Gaussians" [S1].

## Sources
[S1] From Blobs to Spokes: High-Fidelity Surface Reconstruction via Oriented Gaussians, https://doi.org/10.1007/978-3-032-37095-2_4, fetched 2026-10-29
