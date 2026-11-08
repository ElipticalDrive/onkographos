# Optimization Configuration (Adam + Exponential Decay + Selective L2)

Also called: Optimization Procedure, Optimizer Settings, Parameter update rule, Training optimizer setup

## Overview
This configuration defines the standard hyperparameter setup for training neural radiance fields and implicit volume representations, combining the Adam optimizer with a staircase exponential learning rate decay and selective L2 weight decay. It addresses the problem of stabilizing gradient descent and preventing divergence in high-dimensional, ill-conditioned optimization landscapes typical of differentiable volume rendering [S1][S2, unofficial]. It sets itself apart from generic optimizers by explicitly disabling regularization on positional encodings and appearance embeddings while applying it to network weights, and by using a delayed decay schedule to allow initial convergence [S2, unofficial].

## How it works
The method configures the optimization loop by initializing specific optimizers for different parameter groups, defining a learning rate schedule, and applying selective L2 regularization during the update step.

### Inputs and Outputs
*   **Inputs:**
    *   `lr`: Initial learning rate scalar.
    *   `params`: Dictionary of scene parameters (e.g., MLP weights, positional encodings, appearance embeddings).
    *   `loss`: Scalar loss value computed from the rendering equation.
    *   `step`: Current optimization step index.
*   **Outputs:**
    *   `params`: Updated parameters after one optimization step.

### Data Structures
1.  **Optimizer Configuration (`opt_config`):**
    *   `lr_sch`: Learning rate schedule object.
    *   `optimizer_network`: Adam optimizer instance for network weights.
    *   `optimizer_ae`: Adam optimizer instance for appearance embeddings.
    *   `param_labels`: Dictionary mapping parameter names to optimizer targets.
        *   Layout: `{"nerf": "network", "bg": "network", "appearance_embeddings": "ae"}` [S2, unofficial].
    *   `mask`: Boolean mask dictating where L2 decay is applied.
        *   Layout: `{"nerf": {"density_mlp": True, "rgb_mlp": True, "position_encoder": False}, "bg": True, "appearance_embeddings": False}` [S2, unofficial].
    *   `weight_decay`: L2 regularization coefficient. Value: $10^{-6}$ [S2, unofficial].

2.  **Learning Rate Schedule (`optax.exponential_decay`):**
    *   `init_value`: `lr` [S2, unofficial].
    *   `transition_steps`: 10,000 [S2, unofficial].
    *   `decay_rate`: $1/3$ [S2, unofficial].
    *   `staircase`: `True` [S2, unofficial].
    *   `transition_begin`: 10,000 [S2, unofficial].
    *   `end_value`: `lr / 100` [S2, unofficial].

3.  **Adam Optimizer Parameters:**
    *   `b1`: 0.9 [S2, unofficial].
    *   `b2`: 0.99 [S2, unofficial].
    *   `eps`: $10^{-15}$ (for network) or $10^{-8}$ (for appearance embeddings) [S2, unofficial].
    *   `eps_root`: $10^{-15}$ (for network) or $0$ (for appearance embeddings) [S2, unofficial].

### Numbered Steps
1.  **Initialize Learning Rate Schedule:** Create an exponential decay schedule with `init_value=lr`, `transition_steps=10_000`, `decay_rate=1/3`, `staircase=True`, `transition_begin=10_000`, and `end_value=lr/100` [S2, unofficial].
2.  **Initialize Network Optimizer:** Create an Adam optimizer with `learning_rate=lr_sch`, `b1=0.9`, `b2=0.99`, `eps=1e-15`, `eps_root=1e-15` [S2, unofficial].
3.  **Initialize Appearance Embedding Optimizer:** Create an Adam optimizer with `learning_rate=1e-4`, `b1=0.9`, `b2=0.99`, `eps=1e-8`, `eps_root=0` [S2, unofficial].
4.  **Combine Optimizers:** Use `optax.multi_transform` to map parameters to optimizers based on `param_labels` [S2, unofficial].
5.  **Compute Current Learning Rate:** Calculate `current_lr` using the schedule. If `staircase=True`, use integer division for step calculation [S2, unofficial].
6.  **Compute Gradients:** Compute gradients of the loss with respect to all parameters [S1][S2, unofficial].
7.  **Apply Selective L2 Decay:** For each parameter group, apply L2 regularization only if the corresponding mask entry is `True`. The update includes a term $-\lambda \cdot w$ where $\lambda = 10^{-6}$ [S2, unofficial].
8.  **Update Parameters:** Apply the Adam update rule using the computed gradients, current learning rate, and L2 decay terms [S2, unofficial].

### Formulas
The Adam update for a parameter $w$ with gradient $g$ is:
$$
m_t = \beta_1 m_{t-1} + (1 - \beta_1) g_t
$$
$$
v_t = \beta_2 v_{t-1} + (1 - \beta_2) g_t^2
$$
$$
\hat{m}_t = \frac{m_t}{1 - \beta_1^t}, \quad \hat{v}_t = \frac{v_t}{1 - \beta_2^t}
$$
$$
w_{t+1} = w_t - \eta_t \cdot \frac{\hat{m}_t}{\sqrt{\hat{v}_t} + \epsilon} - \lambda \cdot w_t \cdot \mathbb{I}_{\text{mask}}
$$
where $\eta_t$ is the current learning rate, $\lambda = 10^{-6}$ is the weight decay, and $\mathbb{I}_{\text{mask}}$ is 1 if the parameter is masked for decay, 0 otherwise [S2, unofficial].

The learning rate schedule is:
$$
\eta_t = \max\left(\text{end\_value}, \text{init\_value} \cdot \text{decay\_rate}^{\lfloor \frac{t - \text{transition\_begin}}{\text{transition\_steps}} \rfloor}\right)
$$
if `staircase=True`, otherwise continuous [S2, unofficial].

### Parameters
*   `lr`: Initial learning rate. Typical value: $5 \times 10^{-4}$ [S1].
*   `transition_steps`: 10,000 [S2, unofficial].
*   `decay_rate`: $1/3$ [S2, unofficial].
*   `transition_begin`: 10,000 [S2, unofficial].
*   `end_value`: `lr / 100` [S2, unofficial].
*   `weight_decay`: $10^{-6}$ [S2, unofficial].
*   `b1`: 0.9 [S2, unofficial].
*   `b2`: 0.99 [S2, unofficial].
*   `eps`: $10^{-15}$ (network), $10^{-8}$ (appearance) [S2, unofficial].
*   `eps_root`: $10^{-15}$ (network), $0$ (appearance) [S2, unofficial].

### Edge Cases and Pitfalls
*   **Delayed Decay:** `staircase=True` combined with `transition_begin=10_000` delays the first learning rate drop to step 20,000, as the initial LR is held for the first 10k steps before the staircase division triggers [S2, unofficial].
*   **Divergence Risk:** Applying L2 decay to position encoders or appearance embeddings is explicitly disabled to prevent divergence or suboptimal convergence during long training runs [S2, unofficial].
*   **Precision:** Use `eps_root=0` for appearance embeddings to avoid numerical instability when gradients are small [S2, unofficial].

### Pseudocode
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import math

def initialize_optimizer(lr, param_labels, mask):
    """
    Initialize the optimizer configuration.
    """
    # Learning rate schedule
    init_value = lr
    transition_steps = 10000
    decay_rate = 1.0 / 3.0
    staircase = True
    transition_begin = 10000
    end_value = lr / 100.0
    
    # Adam parameters
    b1 = 0.9
    b2 = 0.99
    eps_network = 1e-15
    eps_root_network = 1e-15
    eps_ae = 1e-8
    eps_root_ae = 0.0
    
    # L2 decay
    weight_decay = 1e-6
    
    return {
        'init_value': init_value,
        'transition_steps': transition_steps,
        'decay_rate': decay_rate,
        'staircase': staircase,
        'transition_begin': transition_begin,
        'end_value': end_value,
        'b1': b1,
        'b2': b2,
        'eps_network': eps_network,
        'eps_root_network': eps_root_network,
        'eps_ae': eps_ae,
        'eps_root_ae': eps_root_ae,
        'weight_decay': weight_decay,
        'param_labels': param_labels,
        'mask': mask
    }

def compute_learning_rate(step, config):
    """
    Compute the current learning rate based on the schedule.
    """
    init_value = config['init_value']
    transition_steps = config['transition_steps']
    decay_rate = config['decay_rate']
    staircase = config['staircase']
    transition_begin = config['transition_begin']
    end_value = config['end_value']
    
    if staircase:
        # Integer division for step calculation
        exponent = max(0, (step - transition_begin) // transition_steps)
    else:
        # Continuous decay
        exponent = max(0, (step - transition_begin) / transition_steps)
        
    lr = init_value * (decay_rate ** exponent)
    return max(lr, end_value)

def apply_adam_update(params, grads, step, config, m=None, v=None):
    """
    Apply Adam update with selective L2 decay.
    """
    b1 = config['b1']
    b2 = config['b2']
    weight_decay = config['weight_decay']
    param_labels = config['param_labels']
    mask = config['mask']
    
    lr = compute_learning_rate(step, config)
    
    # Initialize momentum buffers if not provided
    if m is None:
        m = {k: 0.0 for k in params}
    if v is None:
        v = {k: 0.0 for k in params}
        
    updated_params = {}
    
    for key in params:
        p = params[key]
        g = grads[key]
        
        # Determine epsilon based on parameter type
        label = param_labels.get(key, 'network')
        if label == 'ae':
            eps = config['eps_ae']
            eps_root = config['eps_root_ae']
        else:
            eps = config['eps_network']
            eps_root = config['eps_root_network']
            
        # Update momentum buffers
        m[key] = b1 * m[key] + (1 - b1) * g
        v[key] = b2 * v[key] + (1 - b2) * g**2
        
        # Bias correction
        m_hat = m[key] / (1 - b1**step)
        v_hat = v[key] / (1 - b2**step)
        
        # Compute update step
        update = m_hat / (math.sqrt(v_hat) + eps)
        
        # Apply selective L2 decay
        # Check if this parameter should have L2 decay
        should_decay = False
        if key in mask:
            if isinstance(mask[key], dict):
                # Nested mask for network parameters
                # Assume key structure matches mask structure
                # For simplicity, we assume if key is in mask and mask[key] is True, decay
                # In practice, this requires careful mapping of parameter names to mask entries
                # Here we assume a flat mapping for demonstration
                should_decay = mask[key]
            else:
                should_decay = mask[key]
                
        if should_decay:
            update += weight_decay * p
            
        # Update parameter
        updated_params[key] = p - lr * update
        
    return updated_params, m, v

def optimize_step(params, grads, step, config):
    """
    Perform one optimization step.
    """
    new_params, m, v = apply_adam_update(params, grads, step, config)
    return new_params, m, v
```

## Subtopics

### [Gradient Descent Solver](3280-gradient-descent-solver.md)
The Gradient Descent Solver is a matrix-free optimization method used to update voxel grid parameters in differentiable volume rendering, specifically for computing optimal transfer functions. It solves the problem of ill-conditioned linear systems by providing stable behavior for large or nearly rank-deficient problems, serving as a robust alternative to direct solvers like CGLS. It distinguishes itself from its siblings by supporting both constant learning rates and the Adam optimizer, while offering a choice between $l_2$-norm (least squares) and $l_1$-norm (least absolute deviations) loss functions.

### [Training Protocol](4528-training-protocol.md)
The Training Protocol defines the standard optimization procedure for neural networks in volume rendering, encompassing optimizer settings, learning rate scheduling, and data augmentation strategies. It addresses the problem of stabilizing gradient descent and preventing convergence failure during the reconstruction of implicit fields or radiance fields from noisy or sparse inputs. It differs from general optimization loops by explicitly integrating specific regularization priors (such as density priors) and staged training phases (such as pilot networks) to handle early-stage feature integration issues.

#### BFGS Solver with Moreau Envelope
This variant uses the BFGS solver with a Moreau envelope for regularization [S3, unofficial]. It computes the operator norm using `odl.power_method_opnorm`, initializes the Hessian inverse estimate as `0.5 / opnorm ** 2`, and defines the objective as the sum of L2 data discrepancy and a smoothed L1 regularizer [S3, unofficial]. The smoothing parameter `sigma` is 0.03, max iterations are 200, and 10 vectors are stored for limited memory BFGS [S3, unofficial]. A pitfall is that the FBP pseudoinverse must use the 'Hann' filter [S3, unofficial].

#### Inverse Rendering Optimization Setup
This setup uses Adam with L2 loss for inverse rendering of implicit surfaces [S4]. It uses 8 views, a batch size of 2, and runs for 30-90 minutes (64-256 epochs) on an NVIDIA RTX 4090 [S4].

#### Euclidean Projection for Material Constraints
After each Adam update step for material parameters, a Euclidean projection onto the RGB cube $[0,1]^3$ is applied component-wise to ensure physical color values remain valid [S5]. The projection is $\Pi_{[0,1]^3}(c_i) = \max(0, \min(1, c_i))$ for each channel $c_i \in \{r, g, b\}$ [S5]. Initial albedo values are drawn uniformly at random from $[0,1]^3$ [S5].

## Operating characteristics
*   **Time Complexity:** The Adam optimizer has $O(1)$ per parameter per step. The exponential decay schedule is $O(1)$ to compute. Total time per step is linear in the number of parameters [S1][S2, unofficial].
*   **Memory Complexity:** Adam requires storing two momentum buffers per parameter, doubling the memory usage for parameters [S2, unofficial].
*   **Measured Performance:** Convergence takes 9 to 13 hours on a single NVIDIA RTX 3090 GPU for typical scenes [S1]. For inverse rendering, 30 to 90 minutes per optimization on an NVIDIA RTX 4090 [S4].
*   **Strengths:** Stable convergence for ill-conditioned problems; selective L2 decay prevents over-regularization of encodings; delayed decay allows initial exploration [S2, unofficial].
*   **Limitations:** High memory usage due to Adam buffers; requires careful tuning of decay schedule and mask [S2, unofficial].
*   **When to Use:** Standard training for neural radiance fields and implicit volume representations [S1][S2, unofficial].
*   **When Not to Use:** When memory is extremely constrained (use SGD); when position encodings need regularization (disable mask) [S2, unofficial].

## Background
This configuration originates from the standard practices in neural radiance field (NeRF) training, as detailed in [S1] and [S2, unofficial]. It combines the Adam optimizer [Kingma and Ba 2014] with exponential learning rate decay and selective L2 regularization. The specific parameters (e.g., $5 \times 10^{-4}$ LR, $10^{-1}$ decay over $2.5 \times 10^5$ iterations) are from [S1], while the staircase decay and selective mask are from [S2, unofficial]. The method is widely used in differentiable volume rendering for its stability and ease of implementation [S1][S2, unofficial].

## Sources
[S1] Neural RGB-D Surface Reconstruction, https://doi.org/10.1109/cvpr52688.2022.00619, fetched 2026-10-26
[S2] blurgyy/jaxngp, https://github.com/blurgyy/jaxngp, fetched 2026-10-27 (unofficial)
[S3] adler-j/learned_gradient_tomography, https://github.com/adler-j/learned_gradient_tomography, fetched 2026-10-27 (unofficial)
[S4] Path-Space Differentiable Rendering of Implicit Surfaces, https://doi.org/10.1145/3641519.3657473, fetched 2026-10-30
[S5] Optimizing 3D Scene Reconstruction with Differentiable Ray Tracing, https://doi.org/10.31224/5014, fetched 2026-10-29
