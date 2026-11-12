# Training Loop for Refractive Index Field

Also called: Neural Field Training Loop, Refractive Index Field Training Procedure

## Overview
This method defines the optimization loop for training a Multi-Layer Perceptron (MLP) to represent a 3D refractive index (IOR) field and associated emission field, using differentiable ray tracing. It distinguishes itself by combining Eikonal ray tracing with luminance integration, optimizing the IOR field via boundary regularization and luminance reconstruction loss, specifically implemented in JAX for multi-device sharding. It builds on [Training Loop Setup](1522-training-loop-setup.md) and [Training Ray Batching](2252-training-ray-batching.md).

## How it works
**Inputs:**
- `rays`: Batched ray initial conditions (positions and directions).
- `ray_lum_target`: Ground truth luminance values for the rays.
- `X_reg`: Boundary points for regularization.
- `lum_field`: 3D grid of true luminance values for interpolation.
- `s_vals`: Integration sample points along rays.
- `imgs_plane`: Target image plane parameters.
- `plane_locs`: Plane locations for rendering.
- `eta_true`: Ground truth IOR field (for testing/plotting).
- `source_locs`, `covs`: Light source locations and covariance matrices (for emission modeling).

**Outputs:**
- Optimized MLP parameters for `predict_eta` (IOR) and `predict_lum` (emission).
- Checkpoints of the trained model and predicted IOR field (`eta`).
- Training logs (loss, images, IOR slices) via TensorBoard.

**Data Structures:**
- **MLP (`MLP_act`)**:
  - Inputs: Positional encoded coordinates.
  - Hidden layers: Depth `net_depth` (default 4), Width `net_width` (default 128).
  - Activation: ReLU.
  - Skip connections: Enabled (`do_skip=True`).
  - Output: IOR value (scalar).
  - Positional Encoding: Degree `deg_point` (default 4), using `safe_sin` for vectorized sin/cos computation.
- **Optimizer State**: Managed by Optax `Adam`.
- **Loss Function**: Partial application of `lossfn` with parameters `lam` (weight) and `bd_val` (boundary value).

**Numbered Steps:**
1. **Data Preparation**:
   - Load `X_test`, `rays`, `s_vals`, `lum_field`, `ray_lum_target`, `eta_true`.
   - Interpolate `lum_field` to predict emission: `predict_lum = interp_grid(lum_field, cval=0.)`.
   - Compute boundary points: `X_reg = get_X_bd(16)`.
   - Shard data across available JAX devices.

2. **Initialization**:
   - Initialize MLP parameters with `seed=0`.
   - Create Adam optimizer with cosine learning rate decay from `lr_init` to `lr_final` over `num_iters`.
   - Initialize loss function with `lam` (0.1 for gas, 1.0 for NF) and `bd_val` (1.0003 for gas, 1.0 for NF).
   - Initialize render function with `s_vals`, `imgs_plane`, `plane_locs`.

3. **Training Loop** (Iterate `i` from 1 to `num_iters`):
   - **Test/Logging Step** (if `i==1` or `i % test_freq == 0`):
     - Run `test_pstep_and_plot` to compute test loss, max difference, and predicted `eta`.
     - Generate slice plots of `eta` and image comparisons (Model vs Target vs Diff).
     - Log figures to TensorBoard.
   - **Checkpointing** (if `i==1` or `i % save_freq == 0`):
     - Save checkpoint of `opt_state`.
     - Pickle predicted `eta` field to `save_dir/eta_out/eta_out_%04d.p`.
   - **Training Step**:
     - Call `train_pstep` with `rays_shard`, `ray_lum_target_shard`, `X_reg_shard`, `opt_state`, `render_eta`, `lossfn`, `rand_key`.
     - Update `opt_state` and collect gradients.
     - Log training loss.

**Formulas:**
- **Learning Rate Schedule**:
  $$ \text{lr}(x) = \exp\left( \ln(\text{lr\_init}) \cdot \left(1 - \frac{x}{\text{num\_iters}}\right) + \ln(\text{lr\_final}) \cdot \frac{x}{\text{num\_iters}} \right) $$
  Where $x$ is the current iteration.

- **Positional Encoding**:
  $$ \text{posenc}(x, \text{deg}) = \text{concat}\left(x, \sin\left([x \cdot \text{scales}, x \cdot \text{scales} + \frac{\pi}{2}]\right)\right) $$
  Where $\text{scales} = [2^0, 2^1, \dots, 2^{\text{deg}-1}]$.

- **Emission Prediction**:
  $$ \text{emission}(\text{ray\_pos}) = \frac{\exp(\text{logpdf}(\text{ray\_pos}))}{\text{damp}} $$
  Where $\text{logpdf}$ is computed from Gaussian distributions defined by `source_locs` and `covs`.

- **Loss Function**:
  The loss combines luminance reconstruction error and boundary regularization.
  $$ \mathcal{L} = \mathcal{L}_{\text{lum}} + \lambda \cdot \mathcal{L}_{\text{bd}} $$
  Where $\lambda = \text{lam}$ and $\mathcal{L}_{\text{bd}}$ penalizes deviation from `bd_val` at boundary points `X_reg`.

**Parameters:**
- `num_iters`: 50000.
- `batch_size`: 4096.
- `lr_init`: 1e-4.
- `lr_final`: 5e-6.
- `test_freq`: 100.
- `save_freq`: 100.
- `lam`: 0.1 (gas) or 1.0 (nf).
- `bd_val`: 1.0003 (gas) or 1.0 (nf).
- `seed`: 0.

**Edge Cases and Pitfalls:**
- **Precision**: Use `float32` for JAX operations. Ensure `s_vals` and ray positions are within valid bounds for interpolation.
- **Boundaries**: `X_reg` must correctly identify boundary points to apply regularization effectively.
- **Degenerate Input**: If `covs` are singular, `eigh` may fail; ensure positive definite covariance matrices.
- **Memory**: Sharding data across devices is critical for large batches (4096 rays).

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import jax
import jax.numpy as jnp
import optax
from jax.experimental.ode import odeint
import pickle
import tqdm

# Assume helper functions: train_pstep, test_pstep_and_plot, renderfn, posenc, MLP_act, lossfn, get_X_bd, interp_grid, safe_sin

def train_nf(rays, ray_lum_target, lum_field, s_vals, imgs_plane, plane_locs, eta_true, source_locs, covs,
             num_iters=50000, batch_size=4096, lr_init=1e-4, lr_final=5e-6, test_freq=100, save_freq=100,
             lam=0.1, bd_val=1.0003, seed=0, save_dir='./'):
    # 1. Data Preparation
    # Interpolate emission field
    predict_lum = interp_grid(lum_field, cval=0.)
    # Compute boundary points
    X_reg = get_X_bd(16)
    
    # Shard data
    rays_shard = jax.tree_util.tree_map(lambda x: jnp.reshape(x, (jax.device_count(), -1) + x.shape[1:]), rays)
    ray_lum_target_shard = jax.tree_util.tree_map(lambda x: jnp.reshape(x, (jax.device_count(), -1) + x.shape[1:]), ray_lum_target)
    X_reg_shard = jax.tree_util.tree_map(lambda x: jnp.reshape(x, (jax.device_count(), -1) + x.shape[1:]), X_reg)
    
    # 2. Initialization
    key = jax.random.PRNGKey(seed)
    # Initialize MLP parameters
    mlp = MLP_act()
    params = mlp.init(key, jnp.zeros((1, 3)))['params']
    
    # Create optimizer with cosine LR schedule
    def lr_schedule(x):
        return jnp.exp(jnp.log(lr_init) * (1 - x/num_iters) + jnp.log(lr_final) * (x/num_iters))
    optimizer = optax.adam(lr_schedule)
    opt_state = optimizer.init(params)
    
    # Initialize loss function
    lossfn_partial = jax.tree_util.Partial(lossfn, lam=lam, bd_val=bd_val)
    
    # Initialize render function
    render_eta = lambda rays, eta_params: renderfn(rays, lambda x: mlp.apply({'params': eta_params}, x), predict_lum, s_vals)
    
    rand_key = key
    
    # 3. Training Loop
    for i in tqdm.tqdm(range(1, num_iters + 1)):
        # Test/Logging Step
        if i == 1 or i % test_freq == 0:
            start, diff_vmax, eta = test_pstep_and_plot(
                rays, ray_lum_target, eta_true, mlp, opt_state, render_eta, lossfn_partial, rand_key,
                imgs_plane, plane_locs, save_dir, i
            )
        
        # Checkpointing
        if i == 1 or i % save_freq == 0:
            # Save checkpoint
            # Save eta
            eta_out = mlp.apply({'params': params}, jnp.zeros((1, 3))) # Placeholder for actual eta extraction
            with open(f"{save_dir}eta_out/eta_out_{i:04d}.p", 'wb') as f:
                pickle.dump(eta_out, f)
        
        # Training Step
        loss_train, loss2, opt_state, ray_trace, rendering, rand_key, grad = train_pstep(
            rays_shard, ray_lum_target_shard, X_reg_shard, opt_state, render_eta, lossfn_partial, rand_key
        )
        
        # Update parameters
        params = optimizer.update(grad, opt_state) # Note: optax returns new state, params are embedded in state or updated separately depending on implementation
        # In Optax, opt_state contains the updated params if using optimizer.apply_gradients, but here train_pstep likely handles it.
        # Assuming train_pstep returns updated opt_state which contains params.
        # Re-initialize params from opt_state for next iteration if needed by mlp.apply
        # This depends on specific JAX/Optax integration. Standard pattern:
        # updates, opt_state = optimizer.update(grad, opt_state)
        # params = optax.apply_updates(params, updates)
        # However, train_pstep signature suggests it handles the update.
        
    return params, opt_state
```

## Subtopics

### Training Ray Batching
Handles the sampling and sharding of rays for efficient parallel processing. [Training Ray Batching](2252-training-ray-batching.md)

### Training Loop
General framework for differentiable rendering optimization. [Training Loop](4290-training-loop.md)

## Operating characteristics
- **Time Complexity**: Linear with respect to `num_iters` and batch size. Each iteration involves forward pass (ray tracing), loss computation, and backward pass (gradient computation).
- **Memory Complexity**: Proportional to batch size and model size. Sharding helps distribute memory across devices.
- **Measured Performance**: Trained over 50,000 iterations with batch size 4096. Learning rate decays from 1e-4 to 5e-6.
- **Strengths**: Efficient multi-device training via JAX; effective regularization for IOR fields; detailed logging and visualization.
- **Limitations**: Computationally expensive due to ray tracing and gradient computation; sensitive to hyperparameters (lr, lam, bd_val).
- **When to Use**: When training neural fields for refractive index and emission from ray-based measurements.
- **When Not to Use**: For simple volume rendering without IOR optimization or when computational resources are limited.

## Background
This method is based on the `rtnf` repository by Brandon Zhao, which implements differentiable ray tracing for refractive index field estimation. It uses JAX for automatic differentiation and Optax for optimization. The approach combines Eikonal ray tracing with luminance integration to reconstruct 3D fields from 2D measurements.

## Sources
[S1] brandonyzhao/rtnf, https://github.com/brandonyzhao/rtnf, fetched 2026-10-27 (unofficial)
