# Optimization Configuration (Adam + Exponential Decay + Selective L2)

Also called: Parameter update rule, Training optimizer setup

## Overview
This configuration defines the hyperparameter setup and regularization strategy used to optimize neural radiance fields and volume rendering networks [S1, unofficial]. It combines the Adam optimizer with a staircase exponential learning rate decay schedule and selective L2 weight decay applied only to specific network components (MLPs) while excluding positional encodings and appearance embeddings [S1, unofficial]. This setup differs from standard NeRF training loops by explicitly masking L2 regularization to prevent divergence in embedding spaces and by using a decay schedule that holds the initial learning rate for 10,000 steps before decaying by a factor of 1/3 every 10,000 steps [S1, unofficial].

## How it works
### Inputs and Outputs
*   **Inputs**: `lr` (initial learning rate scalar), `step` (current training step integer), `params` (dictionary of network parameters), `param_labels` (mapping parameter prefixes to optimizer targets), `mask` (boolean mask dictating L2 decay application).
*   **Outputs**: Updated `params`, optimizer `state` (momentum estimates, step count).

### Data Structures
*   **`param_labels`**: Dictionary mapping parameter name prefixes to optimizer targets [S1, unofficial].
    *   Layout: `{"nerf": "network", "bg": "network", "appearance_embeddings": "ae"}` [S1, unofficial].
*   **`mask`**: Dictionary defining which parameter groups receive L2 decay [S1, unofficial].
    *   Layout: `{"nerf": {"density_mlp": True, "rgb_mlp": True, "position_encoder": False}, "bg": True, "appearance_embeddings": False}` [S1, unofficial].
    *   Note: `True` indicates L2 decay is applied; `False` indicates it is skipped [S1, unofficial].

### Steps
1.  **Initialize Learning Rate Schedule**: Create an exponential decay scheduler with the parameters defined in the Parameters section [S1, unofficial].
2.  **Initialize Network Optimizer**: Create an Adam optimizer instance using the scheduled learning rate and the specified Adam hyperparameters (`b1`, `b2`, `eps`, `eps_root`) [S1, unofficial].
3.  **Initialize Appearance Embedding Optimizer**: Create a separate Adam optimizer instance with a fixed learning rate of $10^{-4}$ and its specific hyperparameters [S1, unofficial].
4.  **Combine Optimizers**: Use a multi-transform mechanism to map parameter groups to their respective optimizers based on `param_labels` [S1, unofficial].
5.  **Apply L2 Mask**: During the update step, apply L2 weight decay with coefficient `weight_decay` only to parameters where the `mask` is `True` [S1, unofficial].

### Formulas
*   **Learning Rate Schedule**:
    $$
    \text{lr}(t) = \max\left(\text{init\_value} \times \text{decay\_rate}^{\lfloor \frac{\max(0, t - \text{transition\_begin})}{\text{transition\_steps}} \rfloor}, \text{end\_value}\right)
    $$
    where $t$ is the current step [S1, unofficial].
*   **Adam Update**:
    $$
    \begin{aligned}
    m_t &= \beta_1 m_{t-1} + (1 - \beta_1) g_t \\
    v_t &= \beta_2 v_{t-1} + (1 - \beta_2) g_t^2 \\
    \hat{m}_t &= \frac{m_t}{1 - \beta_1^t}, \quad \hat{v}_t = \frac{v_t}{1 - \beta_2^t} \\
    \theta_{t} &= \theta_{t-1} - \text{lr}(t) \frac{\hat{m}_t}{\sqrt{\hat{v}_t} + \text{eps\_root} + \text{eps}}
    \end{aligned}
    $$
    where $g_t$ is the gradient, $\theta$ are parameters, and $t$ is the step count [S1, unofficial].
*   **Selective L2 Decay**:
    $$
    g_t^{\text{modified}} = g_t + \text{weight\_decay} \times \theta_{t-1} \times \text{mask}
    $$
    where `mask` is a boolean array matching the parameter shape [S1, unofficial].

### Parameters
*   **Learning Rate Schedule**: `init_value` = `lr`, `transition_steps` = 10,000, `decay_rate` = 1/3, `staircase` = `True`, `transition_begin` = 10,000, `end_value` = `lr / 100` [S1, unofficial].
*   **Adam Optimizer (Network)**: `learning_rate` = output of schedule, `b1` = 0.9, `b2` = 0.99, `eps` = $10^{-15}$, `eps_root` = $10^{-15}$ [S1, unofficial].
*   **Adam Optimizer (Appearance Embeddings)**: `learning_rate` = $10^{-4}$, `b1` = 0.9, `b2` = 0.99, `eps` = $10^{-8}$, `eps_root` = 0 [S1, unofficial].
*   **L2 Regularization**: `weight_decay` = $10^{-6}$ [S1, unofficial].

### Edge Cases and Pitfalls
*   **Staircase Delay**: With `staircase=True` and `transition_begin=10,000`, the learning rate remains at `lr` for the first 10,000 steps, and the first decay occurs at step 20,000 [S1, unofficial].
*   **L2 Decay on Encodings**: Applying L2 decay to position encoders or appearance embeddings is explicitly disabled to prevent divergence or suboptimal convergence during long training runs [S1, unofficial].

### Pseudocode
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import jax.numpy as jnp
from jax import lax

def compute_learning_rate(step, init_value, transition_steps, decay_rate, staircase, transition_begin, end_value):
    if staircase:
        steps_since_begin = jnp.maximum(0, step - transition_begin)
        num_decays = steps_since_begin // transition_steps
    else:
        num_decays = step / transition_steps
    lr = init_value * (decay_rate ** num_decays)
    return jnp.maximum(lr, end_value)

def adam_step(params, grads, state, lr, b1, b2, eps, eps_root):
    m, v, t = state["m"], state["v"], state["t"]
    t_new = t + 1
    m_new = lax.mul(b1, m) + lax.mul(1 - b1, grads)
    v_new = lax.mul(b2, v) + lax.mul(1 - b2, lax.square(grads))
    m_hat = lax.div(m_new, 1 - b1 ** t_new)
    v_hat = lax.div(v_new, 1 - b2 ** t_new)
    update = lax.mul(lr, lax.div(m_hat, lax.sqrt(v_hat) + eps_root + eps))
    params_new = lax.sub(params, update)
    return params_new, {"m": m_new, "v": v_new, "t": t_new}

def apply_l2_decay(grads, params, mask, weight_decay):
    return lax.add(grads, lax.mul(weight_decay, lax.mul(params, mask)))

def optimize_step(params, grads, state, param_labels, mask, config):
    step = state["t"]
    lr = compute_learning_rate(step, config["init_value"], config["transition_steps"],
                               config["decay_rate"], config["staircase"],
                               config["transition_begin"], config["end_value"])
    
    # Determine target group from param_labels
    group_label = param_labels.get(params["name"], "network")
    
    # Select optimizer hyperparameters
    if group_label == "network":
        lr_opt = lr
        b1_opt, b2_opt, eps_opt, eps_root_opt = 0.9, 0.99, 1e-15, 1e-15
    else:  # "ae"
        lr_opt = 1e-4
        b1_opt, b2_opt, eps_opt, eps_root_opt = 0.9, 0.99, 1e-8, 0.0
        
    # Apply L2 decay if mask is True for this parameter
    mask_val = mask.get(params["name"], False)
    if mask_val:
        grads = apply_l2_decay(grads, params, mask_val, config["weight_decay"])
        
    return adam_step(params, grads, state, lr_opt, b1_opt, b2_opt, eps_opt, eps_root_opt)
```

## Subtopics

### BFGS Solver with Moreau Envelope
A variant used for tomographic reconstruction rather than neural radiance fields [S2, unofficial]. It uses the BFGS quasi-Newton method with a Moreau envelope to smooth the L1 regularizer (GroupL1Norm) for total variation regularization [S2, unofficial]. It requires an initial Hessian inverse estimate based on the operator norm and uses a limited memory approach (`num_store=10`) [S2, unofficial]. The FBP pseudoinverse must use the 'Hann' filter [S2, unofficial].

### [Training Protocol](4528-training-protocol.md)
A simpler protocol used for neural dual contouring and SDF reconstruction. It uses the Adam optimizer and the L2 loss. The number of views is eight, and the batch size is two. The runtime of one optimization is typically 30 to 90 minutes, which usually uses 64 to 256 epochs.

## Operating characteristics
*   **Time Complexity**: Adam is $O(1)$ per parameter per step [G]. The exponential decay calculation is negligible [G].
*   **Memory Complexity**: Adam requires storing two moment estimates per parameter ($2 \times$ parameter size) [G]. The multi-transform setup does not add significant overhead [G].
*   **Performance**: This configuration is designed for long training runs where a slow decay helps stabilize convergence [S1, unofficial]. The runtime is 30 to 90 minutes per optimization on an NVIDIA RTX 4090 GPU, typically spanning 64 to 256 epochs [S3].
*   **Strengths**: Selective L2 decay prevents divergence in embedding spaces; the staircase schedule stabilizes long training runs [S1, unofficial].
*   **Limitations**: The fixed decay schedule requires specific hyperparameter tuning for different datasets; hyperparameters like `eps` are sensitive to the implementation of the Adam optimizer [S1, unofficial].

## Background
This configuration originates from the `jaxngp` repository configuration [S1, unofficial]. It is designed to optimize the MLPs and appearance embeddings in neural radiance fields [S1, unofficial]. The use of selective L2 decay is a known technique in NeRF training to prevent over-regularization of positional encodings, which are critical for high-frequency detail representation [S1, unofficial].

## Sources
[S1] blurgyy/jaxngp, https://github.com/blurgyy/jaxngp, fetched 2026-10-27 (unofficial)
[S2] adler-j/learned_gradient_tomography, https://github.com/adler-j/learned_gradient_tomography, fetched 2026-10-27 (unofficial)
[S3] Path-Space Differentiable Rendering of Implicit Surfaces, https://doi.org/10.1145/3641519.3657473, fetched 2026-10-30
