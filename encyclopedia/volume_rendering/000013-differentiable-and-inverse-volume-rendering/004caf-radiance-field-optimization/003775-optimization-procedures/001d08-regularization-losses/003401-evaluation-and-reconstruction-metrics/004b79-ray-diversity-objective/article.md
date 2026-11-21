# Ray Diversity Objective


## Overview
The Ray Diversity Objective quantifies the variation of rays projected from various viewing poses observing a particular face of an object mesh, aiming to yield better reconstruction by ensuring highly varied novel views [S1, unofficial]. It sets itself apart from sibling metrics by explicitly combining a geometric diversity measure (variance of ray principal axes) with a quality penalty (outlier score based on grazing angles) to guide active view planning [S1, unofficial].

## How it works
**Inputs and Outputs:**
*   **Inputs:** A set of camera poses $\{c_k\}$, a mesh with faces $\{j=1 \dots J\}$, surface normals $\hat{n}_j$ for each face, and for each ray $i$ from each camera $k$, the intersection point or visibility $V_{ijk}$ (binary or continuous weight indicating if ray $i$ from $c_k$ observes face $j$).
*   **Outputs:** The scalar objective value $f_D$.

**Data Structures:**
*   Ray data: For each face $j$, a list of observing rays. Each ray $i$ has a direction vector $\mathbf{r}_i$ (or is defined by its origin and target) and its intersection point on the mesh.
*   Face data: Surface normal $\hat{n}_j$ (unit vector).

**Steps:**

1.  **Compute Augmented Ray-Surface Grazing Angle ($\theta_{ij}$):**
    For each ray $i$ from camera pose $c_k$ observing face $j$:
    $$
    \theta_{ij} = \arccos\left( \frac{\mathbf{r}_i \cdot \hat{n}_j}{|\mathbf{r}_i| |\hat{n}_j|} \right) \frac{180^\circ}{\pi} - 90^\circ
    $$
    where $\mathbf{r}_i$ is the ray direction vector, $\hat{n}_j$ is the surface normal of face $j$, and $|\cdot|$ denotes the Euclidean norm [S1, unofficial].

2.  **Compute Outlier Score ($\psi_{ij}$):**
    Apply soft-thresholding to penalize rays observing the face at extreme grazing angles (approaching orthogonal to the surface normal):
    $$
    \psi_{ij} = 1 - \frac{\alpha_1}{1 + e^{-\theta_{ij} + \alpha_2}}
    $$
    Hyperparameters: $\alpha_1 = 1$, $\alpha_2 = -10$ [S1, unofficial].

3.  **Compute Total Outlier Ratio ($\phi_j$):**
    Calculate the expected outlier score for face $j$, weighted by visibility $V_{ijk}$:
    $$
    K_j = \sum_{i} \sum_{k} V_{ijk}
    $$
    $$
    \phi_j = \frac{1}{K_j} \sum_{i} \sum_{k} \psi_{ij} V_{ijk} = \mathbb{E}_{\mu}[\psi_{ij}]
    $$
    where the sum is over all rays $i$ and camera poses $k$ that observe face $j$ [S1, unofficial].

4.  **Compute Ray Diversity ($\delta_j$):**
    For face $j$, let $I$ be the number of observing rays. Let $x_i, y_i, z_i$ be the components of the principal axis values of each observing ray $i$ (interpreted as vectors from the camera center or normalized direction vectors). Compute the variance of these components:
    $$
    \bar{x} = \frac{1}{I} \sum_{i=1}^I x_i, \quad \bar{y} = \frac{1}{I} \sum_{i=1}^I y_i, \quad \bar{z} = \frac{1}{I} \sum_{i=1}^I z_i
    $$
    $$
    \delta_j = 3 \left( \frac{1}{I} \sum_{i=1}^I (x_i - \bar{x})^2 \right) \left( \frac{1}{I} \sum_{i=1}^I (y_i - \bar{y})^2 \right) \left( \frac{1}{I} \sum_{i=1}^I (z_i - \bar{z})^2 \right)
    $$
    This is equivalent to multiplying the variances of the individual principal axes: $\delta_j = \sigma_x^2 \sigma_y^2 \sigma_z^2$ [S1, unofficial].

5.  **Compute Normalized Effective Ray Diversity ($f_D$):**
    Average the product of diversity and outlier ratio across all $J$ faces:
    $$
    f_D = \frac{1}{J} \sum_{j=1}^J \delta_j \phi_j
    $$
    [S1, unofficial].

**Smoothing and Clipping:**
All objective functions ($f_C, f_Q, f_D, f_T$) are smoothed and clipped using:
$$
\xi(x) = \max\left(0, \min\left(1, \frac{1}{3} \left[ \frac{1}{1 + e^{(-\alpha_3 x) + (\alpha_3 0.5 e)}} + 1 \right] \right)\right)
$$
with $\alpha_3 = 3$ [S1, unofficial].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import numpy as np

def ray_diversity_objective(mesh_faces, rays_per_face, alpha1=1.0, alpha2=-10.0, alpha3=3.0):
    """
    Compute the Ray Diversity Objective.
    
    Args:
        mesh_faces: List of face objects. Each face has:
            - normal: numpy array (3,), unit surface normal n_hat_j
            - rays: List of ray objects. Each ray has:
                - direction: numpy array (3,), ray direction r_i
                - visibility: float V_ijk (weight)
    Returns:
        f_D: float, the normalized effective Ray Diversity objective value.
    """
    J = len(mesh_faces)
    total_score = 0.0
    
    for face in mesh_faces:
        normals = face.normal
        rays = face.rays
        I = len(rays)
        
        if I == 0:
            continue
            
        # Step 1 & 2: Compute outlier scores
        outlier_scores = []
        for ray in rays:
            r_i = ray.direction
            # Ensure direction is normalized for dot product if not already
            r_i_norm = r_i / np.linalg.norm(r_i)
            
            # Cosine of angle between ray and normal
            cos_theta = np.dot(r_i_norm, normals)
            # Clamp to [-1, 1] for numerical stability
            cos_theta = np.clip(cos_theta, -1.0, 1.0)
            
            # Grazing angle in degrees
            theta_deg = np.arccos(cos_theta) * (180.0 / np.pi) - 90.0
            
            # Outlier score psi_ij
            psi = 1.0 - (alpha1 / (1.0 + np.exp(-theta_deg + alpha2)))
            outlier_scores.append(psi)
            
        outlier_scores = np.array(outlier_scores)
        visibilities = np.array([ray.visibility for ray in rays])
        
        # Step 3: Total outlier ratio phi_j
        K_j = np.sum(visibilities)
        if K_j == 0:
            continue
        phi_j = np.sum(outlier_scores * visibilities) / K_j
        
        # Step 4: Ray diversity delta_j
        # Extract principal axis values (x, y, z components of ray directions)
        directions = np.array([ray.direction for ray in rays])
        # Normalize directions if they are not unit vectors
        norms = np.linalg.norm(directions, axis=1, keepdims=True)
        norms = np.where(norms == 0, 1.0, norms) # Avoid division by zero
        unit_directions = directions / norms
        
        x_vals = unit_directions[:, 0]
        y_vals = unit_directions[:, 1]
        z_vals = unit_directions[:, 2]
        
        var_x = np.mean((x_vals - np.mean(x_vals))**2)
        var_y = np.mean((y_vals - np.mean(y_vals))**2)
        var_z = np.mean((z_vals - np.mean(z_vals))**2)
        
        delta_j = 3.0 * var_x * var_y * var_z
        
        # Step 5: Accumulate score
        total_score += delta_j * phi_j
        
    f_D = total_score / J
    
    # Smoothing and clipping
    f_D_smoothed = np.max([0.0, np.min([1.0, (1.0/3.0) * (1.0 / (1.0 + np.exp((-alpha3 * f_D) + (alpha3 * 0.5 * np.e))) + 1.0)])])
    
    return f_D_smoothed
```

## Operating characteristics
*   **Complexity:** Linear in the number of faces and the total number of rays observing those faces.
*   **Strengths:** Explicitly encourages views that provide diverse geometric information (high variance in ray directions) while penalizing views that are geometrically uninformative (grazing angles near 90 degrees).
*   **Limitations:** Requires access to a ground truth mesh to compute surface normals and ray-face intersections, which limits applicability during initial reconstruction phases where the mesh is unknown [S1, unofficial]. This limitation is addressed in the source by using a surrogate network (SOARNet) to estimate the objective [S1, unofficial].
*   **Use Case:** Active view planning for NeRF reconstruction where selecting optimal camera poses is critical [S1, unofficial].

## Background
The Ray Diversity Objective was introduced in the context of SO-NeRF (Surrogate Objectives for NeRF), an active view planning method. It was proposed to address the need for selecting training images from as many views as possible to improve multi-view correspondences [S1, unofficial]. The method is part of a family of surrogate objectives ($f_C, f_Q, f_D, f_T$) used to guide a deep neural network (SOARNet) in predicting next-step camera poses [S1, unofficial].

## Sources
[S1] SO-NeRF: Active View Planning for NeRF using Surrogate Objectives, https://doi.org/10.48550/arXiv.2312.03266, fetched 2026-10-27 (unofficial)
