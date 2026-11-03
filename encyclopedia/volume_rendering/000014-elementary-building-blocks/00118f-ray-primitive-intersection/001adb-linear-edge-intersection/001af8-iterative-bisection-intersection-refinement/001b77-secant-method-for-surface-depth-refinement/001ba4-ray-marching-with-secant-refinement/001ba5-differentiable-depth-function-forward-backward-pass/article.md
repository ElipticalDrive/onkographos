# Differentiable Depth Function Forward/Backward Pass

Also called: DepthFunction.backward, DepthFunction.forward

## Overview
This method implements a PyTorch Autograd Function that wraps the ray marching process, allowing gradients to flow through the depth prediction to the decoder parameters and latent code [S1, unofficial].

## How it works
**Inputs:**
- `ray0`: Ray origins, shape $(N, 3)$.
- `ray_direction`: Unit direction vectors, shape $(N, 3)$.
- `decoder`: Neural network function $f_\phi(p, c)$ evaluating occupancy logits.
- `c`: Latent codes, shape $(N, D)$.
- `n_steps`: Number of ray marching steps.
- `n_secant_steps`: Number of secant refinement steps.
- `tau`: Temperature parameter for soft thresholding.
- `depth_range`: Tuple $(d_{min}, d_{max})$ defining the search interval.
- `method`: String specifying the intersection/refinement method.
- `check_cube_intersection`: Boolean flag for boundary checks.
- `max_points`: Maximum number of points to store.
- `grad_output`: Gradient from the loss function w.r.t. the output depth, shape $(N, 1)$.

**Outputs:**
- `d_pred`: Predicted depth, shape $(N, 1)$.
- `p_pred`: Predicted 3D points, shape $(N, 3)$.
- `mask`: Validity mask, shape $(N, 1)$, where 1 indicates a valid surface hit and 0 indicates no hit.

**Data Structures:**
- `p_pred`: Tensor of shape $(N, 3)$ storing the 3D coordinates of the estimated surface points.
- `d_pred`: Tensor of shape $(N, 1)$ storing the scalar depth values along the ray direction.
- `mask`: Boolean or float tensor of shape $(N, 1)$ indicating successful ray-surface intersection.

**Forward Pass:**
1. Call `perform_ray_marching` with inputs `ray0`, `ray_direction`, `decoder`, `c`, `n_steps`, `n_secant_steps`, `tau`, `depth_range`, `method`, `check_cube_intersection`, and `max_points` to obtain initial `d_pred`, `p_pred`, and `mask` [S1, unofficial].
2. Handle invalid depths by setting $d_{pred}[mask == 0] = \infty$ [S1, unofficial].
3. Handle cases where the first point is not occupied by setting $d_{pred}[mask\_0\_not\_occupied == 0] = 0$ [S1, unofficial].
4. Save `ray0`, `ray_direction`, `d_pred`, `p_pred`, `c`, and `mask` for use in the backward pass [S1, unofficial].

**Backward Pass:**
1. Enable gradient computation for the saved points `p_pred` [S1, unofficial].
2. Evaluate the decoder at the predicted points to get occupancy logits: $f_p = decoder(p_{pred}, c, only\_occupancy=True)$ [S1, unofficial].
3. Compute the gradient of the occupancy logits with respect to the predicted points: $grad\_p = \frac{\partial f_p}{\partial p_{pred}}$ [S1, unofficial].
4. Compute the dot product of this gradient with the ray direction vector: $grad\_p\_dot\_v = \sum_{dim=1}^{3} (grad\_p_{:, dim} \cdot ray\_direction_{:, dim})$ [S1, unofficial].
5. Clamp small gradients to avoid division by zero: if $|grad\_p\_dot\_v| < \epsilon$, set $grad\_p\_dot\_v = \epsilon$, where $\epsilon = 10^{-3}$ [S1, unofficial]. Set invalid mask gradients to 1 [S1, unofficial].
6. Compute the output gradients with respect to the depth prediction: $grad\_outputs = - \frac{grad\_output}{grad\_p\_dot\_v} \cdot mask$ [S1, unofficial].
7. Compute the gradient with respect to the latent code $c$: $grad\_c = \frac{\partial f_p}{\partial c}$ using the computed `grad_outputs` [S1, unofficial].
8. Compute the gradient with respect to the decoder parameters $\phi$: $grad\_\phi = \frac{\partial f_p}{\partial \phi}$ using the computed `grad_outputs` [S1, unofficial].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import torch

class DepthFunction(torch.autograd.Function):
    @staticmethod
    def forward(ctx, ray0, ray_direction, decoder, c, n_steps, n_secant_steps, 
                tau, depth_range, method, check_cube_intersection, max_points, grad_output):
        # [G] The sources do not provide the implementation of perform_ray_marching.
        # It is assumed to return d_pred, p_pred, mask based on the inputs.
        # d_pred, p_pred, mask = perform_ray_marching(...)
        
        # Step 2: Set invalid depths to infinity
        d_pred = torch.zeros_like(ray0[:, :1])
        p_pred = torch.zeros_like(ray0)
        mask = torch.ones_like(d_pred)
        d_pred[mask == 0] = float('inf')
        
        # Step 3: Set invalid depths to 0 if first point not occupied
        # [G] mask_0_not_occupied is not defined in the sources; assumed available as a boolean tensor
        # d_pred[mask_0_not_occupied == 0] = 0
        
        # Step 4: Save for backward pass
        ctx.save_for_backward(ray0, ray_direction, d_pred, p_pred, c, mask)
        ctx.grad_output = grad_output
        ctx.decoder = decoder
        return d_pred, p_pred, mask

    @staticmethod
    def backward(ctx, grad_d_pred, grad_p_pred, grad_mask):
        ray0, ray_direction, d_pred, p_pred, c, mask = ctx.saved_tensors
        grad_output = ctx.grad_output
        decoder = ctx.decoder
        
        # Step 1: Enable gradient computation for p_pred
        p_pred.requires_grad_(True)
        
        # Step 2: Evaluate decoder
        f_p = decoder(p_pred, c, only_occupancy=True)
        
        # Step 3: Compute gradient of f_p w.r.t p_pred
        grad_p = torch.autograd.grad(f_p, p_pred, grad_outputs=torch.ones_like(f_p), create_graph=True)[0]
        
        # Step 4: Compute dot product with ray direction
        grad_p_dot_v = torch.sum(grad_p * ray_direction, dim=1, keepdim=True)
        
        # Step 5: Clamp small gradients
        epsilon = 1e-3
        grad_p_dot_v = torch.where(torch.abs(grad_p_dot_v) < epsilon, torch.full_like(grad_p_dot_v, epsilon), grad_p_dot_v)
        
        # [G] "Set invalid mask gradients to 1" is applied to the denominator for invalid points
        grad_p_dot_v[mask == 0] = 1.0
        
        # Step 6: Compute output gradients
        grad_outputs = - (grad_output / grad_p_dot_v) * mask
        
        # Step 7: Compute gradient w.r.t latent code c
        grad_c = torch.autograd.grad(f_p, c, grad_outputs=grad_outputs, create_graph=True)[0]
        
        # Step 8: Compute gradient w.r.t decoder parameters phi
        # [G] Decoder parameters are tracked by PyTorch autograd; gradients accumulate automatically.
        # The sources do not specify explicit parameter extraction.
        
        return None, None, None, grad_c, None, None, None, None, None, None, None, None
```

## Operating characteristics
Not given in the sources.

## Background
Not given in the sources beyond its implementation in the `autonomousvision/differentiable_volumetric_rendering` repository [S1, unofficial].

## Sources
[S1] autonomousvision/differentiable_volumetric_rendering, https://github.com/autonomousvision/differentiable_volumetric_rendering, fetched 2026-10-26 (unofficial)
