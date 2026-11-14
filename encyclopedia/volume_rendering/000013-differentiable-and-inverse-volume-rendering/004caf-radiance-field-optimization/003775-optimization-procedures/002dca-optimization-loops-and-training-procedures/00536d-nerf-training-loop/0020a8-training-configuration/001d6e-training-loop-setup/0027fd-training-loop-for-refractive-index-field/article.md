# Training Loop for Refractive Index Field

Also called: Neural Field Training Loop, Refractive Index Field Training Procedure

## Overview
This method defines the optimization loop for training a Multi-Layer Perceptron (MLP) to represent a 3D refractive index (IOR) field and associated emission field, using differentiable ray tracing [S1, unofficial]. It distinguishes itself by combining Eikonal ray tracing with luminance integration, optimizing the IOR field via boundary regularization and luminance reconstruction loss, specifically implemented in JAX for multi-device sharding [S1, unofficial]. It builds on [Training Loop Setup](1522-training-loop-setup.md) and [Training Ray Batching](2252-training-ray-batching.md) [G].

## How it works
**Inputs:**
- `rays`: Batched ray initial conditions (positions and directions) [S1, unofficial].
- `ray_lum_target`: Ground truth luminance values for the rays [S1, unofficial].
- `X_reg`: Boundary points for regularization [S1, unofficial].
- `lum_field`: 3D grid of true luminance values for interpolation [S1, unofficial].
- `s_vals`: Integration sample points along rays [S1, unofficial].
- `imgs_plane`: Target image plane parameters [S1, unofficial].
- `plane_locs`: Plane locations for rendering [S1, unofficial].
- `eta_true`: Ground truth IOR field (for testing/plotting) [S1, unofficial].
- `source_locs`, `covs`: Light source locations and covariance matrices for emission modeling [S1, unofficial].

**Outputs:**
- Optimized MLP parameters for `predict_eta` (IOR) and `predict_lum` (emission) [S1, unofficial].
- Checkpoints of the trained model and predicted IOR field (`eta`) [S1, unofficial].
- Training logs (loss, images, IOR slices) via TensorBoard [S1, unofficial].

**Data Structures:**
- **MLP (`MLP_act`)**: Inputs are positional encoded coordinates [S1, unofficial]. Hidden layers have depth `net_depth` (default 4) and width `net_width` (default 128) [S1, unofficial]. Activation is ReLU [S1, unofficial]. Skip connections are enabled (`do_skip=True`) [S1, unofficial]. Output is a scalar IOR value [S1, unofficial]. Positional encoding uses degree `deg_point` (default 4) with vectorized `safe_sin` computation [S1, unofficial].
- **Optimizer State**: Managed by Optax `Adam` with a cosine learning rate schedule [S1, unofficial].
- **Loss Function**: Partial application of `lossfn` with parameters `lam` (weight) and `bd_val` (boundary value) [S1, unofficial].

**Numbered Steps:**
1. **Data Preparation**: Load `X_test`, `rays`, `s_vals`, `lum_field`, `ray_lum_target`, `eta_true` [S1, unofficial]. Interpolate emission: `predict_lum = interp_grid(lum_field, cval=0.)` [S1, unofficial]. Compute boundary points: `X_reg = get_X_bd(16)` [S1, unofficial]. Shard data across available JAX devices [S1, unofficial].
2. **Initialization**: Initialize MLP parameters with `seed=0` [S1, unofficial]. Create Adam optimizer with cosine learning rate decay from `lr_init` to `lr_final` over `num_iters` [S1, unofficial]. Initialize loss function with `lam` (0.1 for gas, 1.0 for NF) and `bd_val` (1.0003 for gas, 1.0 for NF) [S1, unofficial]. Initialize render function with `s_vals`, `imgs_plane`, `plane_locs` [S1, unofficial].
3. **Training Loop** (Iterate `i` from 1 to `num_iters`): If `i==1` or `i % test_freq == 0`, run test step and generate plots [S1, unofficial]. If `i==1` or `i % save_freq == 0`, save checkpoint and pickle `eta` [S1, unofficial]. Call `train_pstep` with `rays_shard`, `ray_lum_target_shard`, `X_reg_shard`, `opt_state`, `render_eta`, `lossfn`, `rand_key` [S1, unofficial]. Log training loss [S1, unofficial].

**Formulas:**
- **Learning Rate Schedule**:
  $$ \text{lr}(x) = \exp\left( \ln(\text{lr\_init}) \cdot \left(1 - \frac{x}{\text{num\_iters}}\right) + \ln(\text{lr\_final}) \cdot \frac{x}{\text{num\_iters}} \right) $$
  Where $x$ is the current iteration [S1, unofficial].

- **Positional Encoding**:
  $$ \text{posenc}(x, \text{deg}) = \text{concat}\left(x, \sin\left([x \cdot \text{scales}, x \cdot \text{scales} + \frac{\pi}{2}]\right)\right) $$
  Where $\text{scales} = [2^0, 2^1, \dots, 2^{\text{deg}-1}]$ [S1, unofficial].

- **Emission Prediction**:
  $$ \text{emission}(\text{ray\_pos}) = \frac{\exp(\text{logpdf}(\text{ray\_pos}))}{\text{damp}} $$
  Where $\text{logpdf}$ is computed from Gaussian distributions defined by `source_locs` and `covs` [S1, unofficial].

- **Loss Function**:
  The loss combines luminance reconstruction error and boundary regularization [S1, unofficial]:
  $$ \mathcal{L} = \mathcal{L}_{\text{lum}} + \lambda \cdot \mathcal{L}_{\text{bd}} $$
  Where $\lambda = \text{lam}$ and $\mathcal{L}_{\text{bd}}$ penalizes deviation from `bd_val` at boundary points `X_reg` [S1, unofficial].

**Parameters:**
- `num_iters`: 50000 [S1, unofficial].
- `batch_size`: 4096 [S1, unofficial].
- `lr_init`: 1e-4 [S1, unofficial].
- `lr_final`: 5e-6 [S1, unofficial].
- `test_freq`: 100 [S1, unofficial].
- `save_freq`: 100 [S1, unofficial].
- `lam`: 0.1 (gas) or 1.0 (nf) [S1, unofficial].
- `bd_val`: 1.0003 (gas) or 1.0 (nf) [S1, unofficial].
- `seed`: 0 [S1, unofficial].

**Edge Cases and Pitfalls:**
- **Precision**: Use `float32` for JAX operations [G]. Ensure `s_vals` and ray positions are within valid bounds for interpolation [G].
- **Boundaries**: `X_reg` must correctly identify boundary points to apply regularization effectively [S1, unofficial].
- **Degenerate Input**: If `covs` are singular, `eigh` may fail; ensure positive definite covariance matrices [S1, unofficial].
- **Memory**: Sharding data across devices is critical for large batches (4096 rays) [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import jax
import jax.numpy as jnp
import optax
import pickle
import tqdm

def safe_sin(x):
    return jnp.sin(jnp.clip(x, -jnp.pi, jnp.pi))  # [G]

def posenc(x, deg):
    if deg == 0:
        return x
    scales = jnp.array([2**i for i in range(deg)])
    xb = jnp.reshape((x[..., None, :] * scales[:, None]), list(x.shape[:-1]) + [-1])
    four_feat = safe_sin(jnp.concatenate([xb, xb + 0.5 * jnp.pi], axis=-1))
    return jnp.concatenate([x] + [four_feat], axis=-1)  # [S1, unofficial]

def interp_grid(grid, cval=0.0):
    def predict_lum(pos):
        return jnp.zeros_like(pos[..., 0]) + cval  # [G]
    return predict_lum  # [G]

def get_X_bd(res):
    coords = jnp.mgrid[0:res, 0:res, 0:res]
    mask = (coords[0] == 0) | (coords[0] == res-1) | \
           (coords[1] == 0) | (coords[1] == res-1) | \
           (coords[2] == 0) | (coords[2] == res-1)
    return coords[:, mask].T  # [G]

def mlp_forward(params, x):
    x_enc = posenc(x, deg_point)
    h = jnp.tanh(jnp.dot(x_enc, params['w1']) + params['b1'])
    h = h + jnp.dot(x_enc, params['w_skip'])  # [S1, unofficial]
    h = jnp.tanh(jnp.dot(h, params['w2']) + params['b2'])
    return h  # [S1, unofficial]

def renderfn(rays, predict_eta, predict_lum, s_vals):
    eta_vals = predict_eta(rays)
    lum_vals = predict_lum(rays)
    return jnp.sum(lum_vals * s_vals, axis=-1)  # [G]

def lossfn(pred, target, eta_boundary, lam, bd_val):
    lum_loss = jnp.mean((pred - target) ** 2)
    bd_loss = jnp.mean((eta_boundary - bd_val) ** 2)
    return lum_loss + lam * bd_loss  # [S1, unofficial]

def train_pstep(rays_shard, ray_lum_target_shard, X_reg_shard, opt_state, render_eta, lossfn_partial, rand_key, optimizer):
    def loss_fn(params):
        pred = render_eta(rays_shard, params)
        eta_boundary = render_eta(X_reg_shard, params)
        return lossfn_partial(pred, ray_lum_target_shard, eta_boundary)
    grad = jax.grad(loss_fn)(opt_state.params)
    updates, new_state = optimizer.update(grad, opt_state)
    new_state = optax.apply_updates(new_state, updates)  # [G]
    return 0.0, 0.0, new_state, None, None, rand_key, grad  # [G]

def test_pstep_and_plot(rays, ray_lum_target, eta_true, mlp, opt_state, render_eta, lossfn_partial, rand_key, imgs_plane, plane_locs, save_dir, i):
    pred = render_eta(rays, opt_state.params)
    eta = mlp.apply({'params': opt_state.params}, jnp.zeros((1, 3)))  # [G]
    # TensorBoard logging omitted for brevity [S1, unofficial]
    return 0.0, 0.0, eta  # [G]

def train_nf(rays, ray_lum_target, lum_field, s_vals, imgs_plane, plane_locs, eta_true, source_locs, covs,
             num_iters=50000, batch_size=4096, lr_init=1e-4, lr_final=5e-6, test_freq=100, save_freq=100,
             lam=0.1, bd_val=1.0003, seed=0, save_dir='./'):
    # 1. Data Preparation
    predict_lum = interp_grid(lum_field, cval=0.0)  # [S1, unofficial]
    X_reg = get_X_bd(16)  # [S1, unofficial]
    rays_shard = jax.tree_util.tree_map(lambda x: jnp.reshape(x, (jax.device_count(), -1) + x.shape[1:]), rays)  # [S1, unofficial]
    ray_lum_target_shard = jax.tree_util.tree_map(lambda x: jnp.reshape(x, (jax.device_count(), -1) + x.shape[1:]), ray_lum_target)  # [S1, unofficial]
    X_reg_shard = jax.tree_util.tree_map(lambda x: jnp.reshape(x, (jax.device_count(), -1) + x.shape[1:]), X_reg)  # [S1, unofficial]

    # 2. Initialization
    key = jax.random.PRNGKey(seed)  # [S1, unofficial]
    mlp = jax.nn.Module(lambda x: x)  # [G]
    params = {'w1': jnp.zeros((3, 128)), 'b1': jnp.zeros(128), 'w_skip': jnp.zeros((3, 128)), 'w2': jnp.zeros((128, 1)), 'b2': jnp.zeros(1)}  # [G]
    
    def lr_schedule(x):
        return jnp.exp(jnp.log(lr_init) * (1 - x/num_iters) + jnp.log(lr_final) * (x/num_iters))  # [S1, unofficial]
    optimizer = optax.adam(lr_schedule)  # [S1, unofficial]
    opt_state = optimizer.init(params)  # [G]

    lossfn_partial = jax.tree_util.Partial(lossfn, lam=lam, bd_val=bd_val)  # [S1, unofficial]
    render_eta = lambda rays, p: renderfn(rays, lambda x: mlp_forward(p, x), predict_lum, s_vals)  # [G]
    rand_key = key  # [G]

    # 3. Training Loop
    for i in tqdm.tqdm(range(1, num_iters + 1)):
        if i == 1 or i % test_freq == 0:
            start, diff_vmax, eta = test_pstep_and_plot(
                rays, ray_lum_target, eta_true, mlp, opt_state, render_eta, lossfn_partial, rand_key,
                imgs_plane, plane_locs, save_dir, i
            )  # [S1, unofficial]
        if (i % save_freq == 0) or (i == 1):
            with open(f"{save_dir}eta_out/eta_out_{i:04d}.p", 'wb') as f:
                pickle.dump(eta, f)  # [S1, unofficial]
        loss_train, loss2, opt_state, ray_trace, rendering, rand_key, grad = train_pstep(
            rays_shard, ray_lum_target_shard, X_reg_shard, opt_state, render_eta, lossfn_partial, rand_key, optimizer
        )  # [S1, unofficial]

    return params, opt_state
```

## Subtopics

### Training Ray Batching
Handles the sampling and sharding of rays for efficient parallel processing. [Training Ray Batching](2252-training-ray-batching.md)

### Training Loop
General framework for differentiable rendering optimization. [Training Loop](4290-training-loop.md)

## Operating characteristics
- **Time Complexity**: Linear with respect to `num_iters` and batch size [S1, unofficial]. Each iteration involves forward pass (ray tracing), loss computation, and backward pass (gradient computation) [G].
- **Memory Complexity**: Proportional to batch size and model size [G]. Sharding helps distribute memory across devices [S1, unofficial].
- **Measured Performance**: Trained over 50,000 iterations with batch size 4096 [S1, unofficial]. Learning rate decays from 1e-4 to 5e-6 [S1, unofficial].
- **Strengths**: Efficient multi-device training via JAX [S1, unofficial]. Effective regularization for IOR fields [S1, unofficial]. Detailed logging and visualization [S1, unofficial].
- **Limitations**: Computationally expensive due to ray tracing and gradient computation [G]. Sensitive to hyperparameters (lr, lam, bd_val) [S1, unofficial].
- **When to Use**: When training neural fields for refractive index and emission from ray-based measurements [S1, unofficial].
- **When Not to Use**: For simple volume rendering without IOR optimization or when computational resources are limited [G].

## Background
This method is based on the `rtnf` repository by Brandon Zhao, which implements differentiable ray tracing for refractive index field estimation [S1, unofficial]. It uses JAX for automatic differentiation and Optax for optimization [S1, unofficial]. The approach combines Eikonal ray tracing with luminance integration to reconstruct 3D fields from 2D measurements [S1, unofficial].

## Sources
[S1] brandonyzhao/rtnf, https://github.com/brandonyzhao/rtnf, fetched 2026-10-27 (unofficial)
