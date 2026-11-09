# Refractive Index Tomography using Neural Fields

Also called: Neural Field Recovery, Single-View Refractive Index Tomography

## Overview
Refractive Index Tomography using Neural Fields (RNTF) is a method to recover a 3D refractive index field ($\eta$) and a luminance field from single-view refracted image measurements by optimizing neural fields that represent these quantities [S1, unofficial]. It differs from standard NeRF-like methods by modeling the physical propagation of light through a medium with varying refractive index, requiring the integration of ray trajectories via Ordinary Differential Equations (ODEs) rather than fixed-step volumetric integration [S1, unofficial].

## How it works
**Inputs and Outputs:**
*   **Inputs:** A single-view refracted image (or simulated measurement), camera parameters (field of view `fov_angle`, resolution `res`), and an initial refractive index guess `n_init` [S1, unofficial].
*   **Outputs:** A trained neural network representing the refractive index field $\eta(\mathbf{x})$ and optionally a luminance field $L(\mathbf{x})$ [S1, unofficial].

**Data Structures:**
*   **Ray Input Array:** Shape `[batch_size, 6 + num_aux]` [S1, unofficial]. The first 3 elements are the ray position $(x, y, z)$, the next 3 are the ray direction $(dx, dy, dz)$, and the remaining elements are auxiliary data [S1, unofficial].
*   **ODE State $y$:** Shape `[batch_size, state_dim]` [S1, unofficial]. `state_dim` is 6 for path-only tracing (position + direction) or 7 for luminance tracing (position + direction + luminance) [S1, unofficial].
*   **Neural Field:** A Multi-Layer Perceptron (MLP) taking 3D coordinates as input and outputting the refractive index $\eta$ (and optionally its gradient $\nabla \eta$) [S1, unofficial].

**Numbered Steps:**

1.  **Ray Initialization:** Initialize rays with starting positions (`ap_pts`) and directions (`rays_d`) derived from camera parameters (`fov_angle`, `res`) [S1, unofficial]. The ray direction is normalized and scaled by the initial refractive index $n_{init}$ at the starting position to ensure correct physical propagation, as the ODE uses `ray_dir / eta` [S1, unofficial].
2.  **Define ODE State:** Construct the initial state $y_0$ by concatenating ray position, ray direction, and auxiliary data (e.g., initial luminance) [S1, unofficial].
3.  **Define ODE Function:** Define the derivative function $f(y, s)$ which computes the derivatives with respect to path length $s$:
    *   $\frac{d}{ds}\mathbf{r}_{pos} = \frac{\mathbf{r}_{dir}}{\eta(\mathbf{r}_{pos})}$ [S1, unofficial].
    *   $\frac{d}{ds}\mathbf{r}_{dir} = \nabla \eta(\mathbf{r}_{pos})$ [S1, unofficial].
    *   $\frac{d}{ds}L = L_{emission}(\mathbf{r}_{pos})$ for path-only tracing, or $\frac{d}{ds}L = \frac{L_{emission}}{\eta^2}$ for radiance adjustment [S1, unofficial].
4.  **Integrate ODE:** Integrate the ODE using an ODE solver (e.g., `jax.experimental.ode.odeint`) over specified path length values `s_vals` [S1, unofficial].
5.  **Extract Results:** Extract the final ray positions (`ray_trace`) and integrated luminance (`ray_lum`) from the integration result $y$ [S1, unofficial].
6.  **Optimize:** Optimize the neural field parameters using gradient descent via automatic differentiation (e.g., `jax.value_and_grad` and `apply_gradients`) to minimize the difference between rendered and target images [S1, unofficial].

**Formulas:**
The ODE derivatives are defined as:
$$
\frac{d}{ds}\mathbf{r}_{pos} = \frac{\mathbf{r}_{dir}}{\eta}
$$
$$
\frac{d}{ds}\mathbf{r}_{dir} = \nabla \eta
$$
$$
\frac{d}{ds}L = \frac{L_{emission}}{\eta^2} \quad \text{(for radiance adjustment)}
$$
where $\mathbf{r}_{pos}$ is the ray position, $\mathbf{r}_{dir}$ is the ray direction, $\eta$ is the refractive index, $\nabla \eta$ is the gradient of the refractive index, $L$ is the luminance, and $L_{emission}$ is the local emission [S1, unofficial].

**Parameters:**
*   `s_vals`: Path length values at which to evaluate the ray-traced path integral [S1, unofficial].
*   `lam`: Regularization weight for the boundary term in the loss function [S1, unofficial].
*   `res`: Resolution of the ray grid, determining the number of rays ($res^2$) [S1, unofficial].
*   `fov_angle`: Field of view angle in degrees, used to calculate the aperture points and ray directions [S1, unofficial].
*   `n_init`: Initial refractive index at the starting position [S1, unofficial].

**Pitfalls:**
*   Ray direction magnitude must be scaled by the initial refractive index $n_{init}$ to ensure correct physical propagation [S1, unofficial].
*   For dark matter simulation, ODE integration uses tight tolerances (`rtol=1e-10`, `atol=1e-10`) to ensure precision [S1, unofficial].
*   The luminance integrand must be adjusted by dividing by $\eta^2$ if radiance adjustment is required, which distinguishes `renderfn_rad` from `renderfn` [S1, unofficial].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import jax
import jax.numpy as jnp

def ode_fn_lum(y, s, predict_eta_grad_fn, predict_lum):
    """
    Computes the derivatives for the ODE state y.
    
    Args:
        y: ODE state [batch_size, 7] (pos, dir, lum)
        s: Path length (unused in autonomous ODE)
        predict_eta_grad_fn: Function that takes positions and returns (eta, eta_grad)
        predict_lum: Function that takes positions and returns luminance emission
        
    Returns:
        dy: Derivatives [batch_size, 7]
    """
    ray_pos = y[..., :3]
    ray_dir = y[..., 3:6]
    
    # Predict refractive index and its gradient
    eta, eta_grad = predict_eta_grad_fn(jnp.reshape(ray_pos, [-1, 3]))
    eta = jnp.reshape(eta, [-1, 1])
    eta_grad = jnp.reshape(eta_grad, [-1, 3])
    
    # Predict luminance emission
    lum = predict_lum(jnp.reshape(ray_pos, [-1, 3]))
    lum = jnp.reshape(lum, [-1, 1])
    
    # Compute derivatives
    d_ds_ray_pos = ray_dir / eta
    d_ds_ray_dir = eta_grad
    # Radiance adjustment: divide luminance by eta^2
    d_ds_ray_lum = lum / (eta ** 2)
    
    out = jnp.concatenate([d_ds_ray_pos, d_ds_ray_dir, d_ds_ray_lum], axis=-1)
    return out

def renderfn_rad(rays, predict_eta_grad_fn, predict_lum, s_vals):
    """
    Renders refracted images by integrating the ODE.
    
    Args:
        rays: Ray input array [batch_size, 6 + num_aux]
        predict_eta_grad_fn: Function to predict eta and its gradient
        predict_lum: Function to predict luminance emission
        s_vals: Path length values for integration
        
    Returns:
        ray_trace: Final ray positions [batch_size, 3]
        ray_lum: Integrated luminance [batch_size, 1]
    """
    # Initialize ODE state y_0
    # Assuming rays has shape [batch_size, 6 + num_aux]
    # First 3: pos, Next 3: dir, Rest: aux
    ray_pos = rays[..., :3]
    ray_dir = rays[..., 3:6]
    # Initialize luminance to 0; source does not specify exact initialization [G]
    ray_lum_init = jnp.zeros_like(ray_pos[..., :1])
    
    y_0 = jnp.concatenate([ray_pos, ray_dir, ray_lum_init], axis=-1)
    
    # Define the ODE function
    def ode_fn(y, s):
        return ode_fn_lum(y, s, predict_eta_grad_fn, predict_lum)
    
    # Integrate ODE
    # y shape: [len(s_vals), batch_size, state_dim]
    y = jax.experimental.ode.odeint(ode_fn, y_0, s_vals)
    
    # Extract final state
    y_final = y[-1]
    ray_trace = y_final[..., :3]
    ray_lum = y_final[..., 6:7]
    
    return ray_trace, ray_lum
```

## Operating characteristics
*   **Time Complexity:** Depends on the number of ODE integration steps and the complexity of the neural network evaluation per step [S1, unofficial]. Tight tolerances (e.g., `rtol=1e-10`, `atol=1e-10`) increase the number of steps and thus computation time [S1, unofficial].
*   **Strengths:** Physically accurate modeling of light propagation in refractive media; capable of recovering smooth refractive index fields and luminance fields from single views [S1, unofficial].
*   **Limitations:** Requires careful tuning of ODE solver tolerances [S1, unofficial].
*   **When to use:** Gas flow recovery, smooth refractive field recovery, and dark matter halo recovery simulations [S1, unofficial].

## Background
Refractive Index Tomography using Neural Fields was introduced by Brandon Zhao et al. in the paper "Single-View Refractive Index Tomography using Neural Fields" (arXiv:2309.04437) [S1, unofficial]. The method leverages JAX for automatic differentiation and efficient ODE integration to optimize neural fields representing the refractive index and luminance fields [S1, unofficial]. It is used for applications such as gas flow visualization, smooth refractive field reconstruction, and simulating dark matter halos [S1, unofficial].

## Sources
[S1] brandonyzhao/rtnf, https://github.com/brandonyzhao/rtnf, fetched 2026-10-27 (unofficial)
