# Ray-Conditioned Sample Prediction with Geometric Primitives

Also called: E_phi network, HyperReel sampling

## Overview
This method predicts sample locations for volume rendering by intersecting input rays with predicted geometric primitives, ensuring multi-view consistency and efficient sampling [S1]. It differs from sibling methods like [Occupancy Proposal Sampling](14956-occupancy-proposal-sampling.md) by explicitly modeling sample positions as intersections with world-space primitives (planes or spheres) rather than discrete probability distributions or occupancy grids [S1].

## How it works
**Inputs and Outputs:**
*   **Input:** Ray origin $\mathbf{o}$ and direction $\vec{\omega}$.
*   **Output:** A set of $n$ sample points $\{\mathbf{x}_k\}_{k=1}^n$ and associated scalar values $\{\delta_k\}_{k=1}^n$ used to compute weights.

**Data Structures:**
*   **Ray Representation:** Plücker coordinates $\mathbf{r} = (\vec{\omega}, \vec{\omega} \times \mathbf{o})$ [S1].
*   **Network $E_\phi$:** A 6-layer Multi-Layer Perceptron (MLP) with 256 hidden units and Leaky ReLU activations [S1].
*   **Geometric Primitives $G_k$:** Defined in the world coordinate frame. The network outputs parameters defining these primitives based on the input ray [S1].
    *   For forward-facing scenes: $n=32$ axis-aligned z-planes [S1].
    *   For other scenes: $n=32$ concentric spherical shells centered at the origin [S1].
*   **Offsets:** Per-sample displacement vectors $\mathbf{e}_k$ [S1].
*   **Scalars:** Per-sample scalar values $\delta_k$ [S1].

**Steps:**
1.  **Reparameterize Ray:** Convert ray origin $\mathbf{o}$ and direction $\vec{\omega}$ into Plücker coordinates:
    $$ \mathbf{r} = \text{Plucker}(\mathbf{o}, \vec{\omega}) = (\vec{\omega}, \vec{\omega} \times \mathbf{o}) $$
    [S1]
2.  **Predict Primitives:** Pass $\mathbf{r}$ through network $E_\phi$ to predict parameters for $n$ geometric primitives $\{G_1, \dots, G_n\}$ [S1].
3.  **Intersect Ray:** Compute initial sample points $\mathbf{x}_k$ by intersecting the ray with each primitive $G_k$:
    $$ \mathbf{x}_k = \text{inter}(G_k; \mathbf{o}, \vec{\omega}) $$
    [S1]
    *Note: $\text{inter}$ is a differentiable operation [S1].*
4.  **Predict Offsets and Scalars:** The network also outputs per-sample displacement vectors $\mathbf{e}_k$ and scalar values $\delta_k$ [S1].
5.  **Compute Displacement:** Calculate the final displacement $\mathbf{d}_k$ by scaling the offset $\mathbf{e}_k$ with the sigmoid of $\delta_k$:
    $$ \mathbf{d}_k = \gamma(\delta_k) \mathbf{e}_k $$
    where $\gamma$ is the sigmoid operator [S1].
6.  **Update Samples:** Add the displacement to the initial intersection points:
    $$ \mathbf{x}_k^{\text{new}} = \mathbf{x}_k + \mathbf{d}_k $$
    [S1]

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import torch
import torch.nn as nn
import torch.nn.functional as F

class RayConditionedSampler(nn.Module):
    def __init__(self, num_primitives=32, scene_type='forward'):
        super().__init__()
        self.num_primitives = num_primitives
        self.scene_type = scene_type
        # 6-layer MLP, 256 hidden units, Leaky ReLU
        # Outputs: primitive params (varies by type), offsets (3 * num_primitives), scalars (num_primitives)
        self.net = nn.Sequential(
            nn.Linear(6, 256), nn.LeakyReLU(),
            nn.Linear(256, 256), nn.LeakyReLU(),
            nn.Linear(256, 256), nn.LeakyReLU(),
            nn.Linear(256, 256), nn.LeakyReLU(),
            nn.Linear(256, 256), nn.LeakyReLU(),
            nn.Linear(256, 3 * num_primitives + num_primitives + self._get_primitive_dim())
        )

    def _get_primitive_dim(self):
        if self.scene_type == 'forward':
            return 1 # z-plane position
        else:
            return 1 # sphere radius

    def forward(self, ray_origins, ray_directions):
        # 1. Reparameterize to Plucker coordinates
        # r = (omega, omega x o)
        cross = torch.cross(ray_directions, ray_origins, dim=-1)
        plucker = torch.cat([ray_directions, cross], dim=-1) # Shape: (..., 6)

        # 2. Predict network outputs
        # Note: In practice, primitive params might be shared or ray-dependent.
        # Based on [S1], E_phi takes ray as input.
        # We assume the network outputs primitive parameters, offsets, and scalars.
        # For simplicity, assuming primitive params are also ray-dependent or fixed per scene type.
        # The source implies E_phi(o, w) -> (G1...Gn, e1...en, delta1...deltan).
        # Let's assume the last layer outputs primitive params + offsets + scalars.
        
        # Adjust input dim if primitive params are ray-dependent. 
        # If primitives are fixed per scene (e.g. just z-planes at fixed z), 
        # the network might only predict offsets/scalars. 
        # However, [S1] says "predict a set of geometric primitives... defined in world coordinate frame".
        # And "primitive parameters themselves are a function of the input ray".
        # So we predict primitive params too.
        
        out = self.net(plucker)
        
        # Split outputs
        prim_dim = self._get_primitive_dim()
        prim_params = out[..., :self.num_primitives * prim_dim]
        offsets = out[..., self.num_primitives * prim_dim : self.num_primitives * prim_dim + self.num_primitives * 3]
        scalars = out[..., self.num_primitives * prim_dim + self.num_primitives * 3 :]
        
        # Reshape
        prim_params = prim_params.view(-1, self.num_primitives, prim_dim)
        offsets = offsets.view(-1, self.num_primitives, 3)
        scalars = scalars.view(-1, self.num_primitives)

        # 3. Intersect ray with primitives to get initial points
        # This depends on primitive type.
        if self.scene_type == 'forward':
            # Axis-aligned z-planes. Plane equation: z = z_k
            # Ray: o + t * w. Intersection: t = (z_k - o_z) / w_z
            z_planes = prim_params[..., 0] # Shape: (..., num_primitives)
            # Avoid division by zero
            w_z = ray_directions[..., 2:3]
            t = (z_planes - ray_origins[..., 2:3]) / w_z
            initial_points = ray_origins.unsqueeze(1) + t.unsqueeze(-1) * ray_directions.unsqueeze(1)
        else:
            # Concentric spherical shells. Sphere: |x|^2 = r_k^2
            # Ray: o + t * w. Intersection: |o + t*w|^2 = r^2 => t^2 + 2(o.w)t + |o|^2 - r^2 = 0
            radii = prim_params[..., 0] # Shape: (..., num_primitives)
            radii_sq = radii ** 2
            o_dot_w = torch.sum(ray_origins * ray_directions, dim=-1, keepdim=True) # (..., 1)
            o_sq = torch.sum(ray_origins ** 2, dim=-1, keepdim=True) # (..., 1)
            
            # Quadratic formula: t = (-B - sqrt(B^2 - 4AC)) / 2A, where A=1, B=2(o.w), C=|o|^2 - r^2
            # We want the closest intersection in front of the camera (t > 0)
            # Discriminant
            disc = 4 * o_dot_w ** 2 - 4 * (o_sq - radii_sq)
            # Clamp disc to avoid NaNs
            disc = torch.clamp(disc, min=0.0)
            sqrt_disc = torch.sqrt(disc)
            
            # Two solutions
            t1 = (-2 * o_dot_w - sqrt_disc) / 2.0
            t2 = (-2 * o_dot_w + sqrt_disc) / 2.0
            
            # Choose the smallest positive t
            # If both negative, pick the least negative (closest to origin)? Or ignore?
            # Standard ray marching usually picks the first hit.
            # Let's pick the smallest t that is > 0. If none, pick smallest t.
            t = torch.where(t1 > 0, t1, t2)
            t = torch.where(t2 > 0, t2, t) # This logic is flawed if both > 0. 
            # Correct logic:
            t_candidates = torch.stack([t1, t2], dim=-1) # (..., num_primitives, 2)
            # Filter for t > 0
            valid = t_candidates > 0
            # If no valid, use min(t1, t2)
            t_min = torch.min(t1, t2)
            t = torch.where(valid.any(dim=-1), t_candidates[valid].view(t_candidates.shape[0], -1).min(dim=-1)[0], t_min)
            
            initial_points = ray_origins.unsqueeze(1) + t.unsqueeze(-1) * ray_directions.unsqueeze(1)

        # 4. Compute displacement
        # d_k = sigmoid(delta_k) * e_k
        weights = torch.sigmoid(scalars) # (..., num_primitives)
        weights = weights.unsqueeze(-1) # (..., num_primitives, 1)
        displacements = weights * offsets # (..., num_primitives, 3)

        # 5. Update samples
        final_points = initial_points + displacements

        return final_points, scalars
```

**Parameters:**
*   **Network Architecture:** 6-layer MLP, 256 hidden units, Leaky ReLU [S1].
*   **Number of Primitives:** 32 [S1].
*   **Initialization:** Scalar values $\delta_k$ should be initialized to negative values to keep the sigmoid output close to 0 and gradients small, preventing premature unmasking of point offsets [S1].

**Edge Cases and Pitfalls:**
*   **Initialization:** Failing to initialize $\delta_k$ to negative values may cause the network to unmask point offsets prematurely during training, leading to instability [S1].
*   **Ray-Primitive Intersection:** The intersection operation must be differentiable [S1]. Degenerate rays (e.g., $w_z=0$ for z-planes) require handling to avoid division by zero [G].

## Operating characteristics
*   **Time Complexity:** The method requires a single forward pass through the network $E_\phi$ per ray, followed by $n$ intersection calculations. This is significantly faster than importance sampling methods that require hundreds or thousands of queries [S1].
*   **Memory Complexity:** Depends on the network size (6-layer MLP) and the storage of primitive parameters and offsets.
*   **Performance:** Enables real-time rendering with low sample counts [S1].
*   **Strengths:** Multi-view consistency is inherent because primitives are defined in world space; if two rays observe the same point, the network predicts the same primitive [S1]. Flexible modeling of complex view-dependent appearance, such as distorted refractions [S1].
*   **Limitations:** The initial sample points lie along the ray, which may be less flexible than other methods [S1].

## Background
*   **Origin:** Introduced in "HyperReel: High-Fidelity 6-DoF Video with Ray-Conditioned Sampling" [S1].
*   **History:** Developed to address the high sample counts required by traditional importance sampling in volume rendering [S1].
*   **Usage:** Used for high-fidelity 6-DoF video synthesis, including static and dynamic scenes [S1].

## Sources
[S1] HyperReel: High-Fidelity 6-DoF Video with Ray-Conditioned Sampling, https://doi.org/10.1109/cvpr52729.2023.01594, fetched 2026-10-26
