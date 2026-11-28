# Canonical Neural Radiance Field for X-ray Tomography

Also called: Canonical volume fitting, NeRF for X-ray

## Overview
Canonical Neural Radiance Field for X-ray Tomography is a method that reconstructs the 3D density field of a static object (canonical volume) using a Neural Radiance Field (NeRF), trained on rich projection data to serve as a prior for dynamic reconstruction [S1, unofficial]. It differs from full spatio-temporal networks by decoupling the static canonical volume representation from the time-dependent deformation field, addressing the poor performance observed when simply adding time as a direct input to conventional NeRFs [S1, unofficial].

## How it works
The method relies on the physical principle of X-ray attenuation and the mathematical framework of the Radon transform [S1, unofficial].

**Inputs and Outputs:**
*   **Inputs:** Rich projection data (sinograms) acquired at specific time points (typically $t=0$ and $t=1$) [S1, unofficial].
*   **Outputs:** A canonical volume representation defined by a neural network mapping canonical coordinates $(x_0, y_0, z_0)$ to density $\sigma$ and color $c$ [S1, unofficial].

**Physical and Mathematical Background:**
X-ray attenuation follows the Beer-Lambert law [S1, unofficial]:
$$
I = I_0 \exp \left( - \int \mu(x) dx \right)
$$
where $I$ is the transmitted intensity, $I_0$ is the initial intensity, and $\mu(x)$ is the attenuation coefficient [S1, unofficial]. The projection $p_\theta(r)$ at angle $\theta$ is the Radon transform of the density function $\mu(x,y)$ [S1, unofficial]:
$$
p_\theta(r) = \int_{x,y} \mu(x,y) \delta(x \cos \theta + y \sin \theta - r) dx dy
$$
Tomographic reconstruction is the inverse operation of this transform [S1, unofficial].

**Data Structures:**
The canonical volume is encoded by a neural network $\mathcal{N}$ [S1, unofficial]. The network takes canonical 3D coordinates $\mathbf{x}_0 = (x_0, y_0, z_0)$ as input and outputs density $\sigma$ and color $c$ [S1, unofficial]. The specific architecture (e.g., number of layers, hidden dimensions) is not detailed in the sources [G].

**Algorithm Steps:**
1.  **Data Acquisition:** Acquire rich, many-projection X-ray data for the object at static time points, specifically $t=0$ and $t=1$ [S1, unofficial].
2.  **Pre-training (Canonical Volume Fitting):** Train the neural network $\mathcal{N}$ to minimize the difference between the rendered projections and the measured sinograms [S1, unofficial].
3.  **Parameter Selection:** Ensure the number of projections is sufficient to satisfy the sampling criterion for the sample [S1, unofficial]. A typical value is 3000 projections, ensuring the maximum displacement of sample points between projections is within one voxel [S1, unofficial].
4.  **Deformation Field Calculation (Contextual):** While the canonical volume is fitted using rich data, the deformation field is subsequently calculated from sparse few-projection data at intermediate times $t$ [S1, unofficial]. This step is part of the broader spatio-temporal model but distinct from the canonical volume fitting itself [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def train_canonical_volume(projections, angles, network, optimizer, num_epochs):
    for epoch in range(num_epochs):
        for projection, angle in zip(projections, angles):
            # 1. Sample rays through the canonical volume
            def generate_rays(proj_shape, angle):
                origins, directions = [], []
                for i in range(proj_shape[0]):
                    for j in range(proj_shape[1]):
                        origins.append([i, j, 0.0])
                        directions.append([0.0, 0.0, 1.0])
                return origins, directions

            # 2. Render the projection using the current network
            def render_volume(net, origins, directions):
                rendered = []
                for o, d in zip(origins, directions):
                    points = [[o[k] + t * d[k] for k in range(3)] for t in [0.1, 0.2, 0.3, 0.4, 0.5]]
                    densities = [net(p) for p in points]
                    rendered.append(sum(densities) * 0.1)
                return rendered

            # 3. Compute loss against measured projection
            def mean_squared_error(pred, target):
                return sum((p - t)**2 for p, t in zip(pred, target)) / len(target)

            rays_o, rays_d = generate_rays(projection.shape, angle)
            rendered_proj = render_volume(network, rays_o, rays_d)
            loss = mean_squared_error(rendered_proj, projection.flatten())

            # 4. Backpropagate to update network weights
            optimizer.zero_grad()
            loss.backward()
            optimizer.step()
```

**Edge Cases and Pitfalls:**
*   **Insufficient Projections:** Using too few projections violates the requirement that maximum displacement be within one voxel, leading to reconstruction artifacts [S1, unofficial].
*   **Time as Direct Input:** Simply adding time $t$ as an input to the NeRF $(x, y, z, t) \rightarrow (c, \sigma)$ yields poor performance; the decoupled canonical approach is necessary [S1, unofficial].

## Operating characteristics
*   **Time Complexity:** Not given in the sources [S1, unofficial].
*   **Memory Complexity:** Not given in the sources [S1, unofficial].
*   **Strengths:** More efficient than conventional reconstruction methods for certain data modalities; enables 3D reconstruction during dynamic events by leveraging static priors [S1, unofficial].
*   **Limitations:** Requires rich projection data at static time points ($t=0, t=1$) for pre-training; not suitable for scenes with topological changes that cannot be modeled by deformation fields [S1, unofficial].
*   **When to use:** When reconstructing dynamic scenes where topology is preserved, and rich data is available at key static frames [S1, unofficial].
*   **When not to use:** When only sparse data is available at all time points, or when the scene undergoes topological changes that break the deformation field assumption [S1, unofficial].

## Background
The method was introduced in the paper "Neural rendering enables dynamic tomography" [S1, unofficial]. It addresses the challenge of 3D reconstruction during dynamic events by shifting from conventional CT to neural rendering techniques [S1, unofficial]. The work demonstrates that NeRFs can reconstruct data modalities more efficiently than conventional methods and develops a spatio-temporal model with spline-based deformation fields for real-world experiments [S1, unofficial].

## Sources
[S1] Neural rendering enables dynamic tomography, https://doi.org/10.48550/arxiv.2410.20558, fetched 2026-10-26 (unofficial)
