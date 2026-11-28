# Partially Learned Gradient Descent

Also called: Learned Gradient Descent, Learned iterative reconstruction, Partially learned gradient descent without regularizer, Unrolled Optimization with Learned Updates, Unrolled optimization

## Overview
Partially Learned Gradient Descent (PLGD) is an iterative reconstruction method that unrolls a fixed number of optimization steps into a neural network [S1, unofficial]. It initializes the reconstruction with a filtered backprojection (FBP) estimate and iteratively refines it by passing the current estimate, gradients, and a momentum memory state through a convolutional network to compute the next update [S1, unofficial]. Unlike standard gradient descent which uses a fixed or manually tuned step size, PLGD learns the optimization dynamics (step size and momentum) directly from data, while retaining the explicit physical forward model (e.g., RayTransform) to ensure interpretability and robustness to distribution shifts [S1, unofficial]. It differs from fully learned end-to-end CNNs by preserving the gradient structure of the inverse problem [S1, unofficial].

## How it works
**Inputs and Outputs:**
*   **Input:** Noisy measurement data $y$ (e.g., sinograms), initial estimate $x_0$ (typically FBP of $y$), and the forward operator $\mathcal{A}$ (and its adjoint $\mathcal{A}^*$) [S1, unofficial].
*   **Output:** Reconstructed volume $x$ [S1, unofficial].
*   **Parameters:** Number of iterations $n_{iter}$, number of memory channels $n_{memory}$, network weights $W_1, b_1, W_2, b_2, W_3, b_3$ [S1, unofficial].

**Data Structures:**
*   **State Tensor $x$:** Shape `[batch, height, width, 1]` [S1, unofficial].
*   **Memory Tensor $s$:** Shape `[batch, height, width, n_memory]`, initialized to zeros [S1, unofficial].
*   **Network Weights:**
    *   $W_1$: `[3, 3, C_{in}, 32]`, $b_1$: `[1, 1, 1, 32]`
    *   $W_2$: `[3, 3, 32, 32]`, $b_2$: `[1, 1, 1, 32]`
    *   $W_3$: `[3, 3, 32, n_{memory} + 1]`, $b_3$: `[1, 1, 1, n_{memory} + 1]`
    *   Where $C_{in}$ is the number of input channels to the network (2 for "No Regularizer" variant, 3 for "With Regularizer" variant) [S1, unofficial].

**Algorithm Steps:**
1.  **Initialization:**
    *   Compute $x_0 = \text{FBP}(y)$ [S1, unofficial].
    *   Initialize $s = 0$ with shape `[batch, height, width, n_memory]` [S1, unofficial].
    *   Normalize the forward operator $\mathcal{A}$ and its pseudoinverse by the operator norm $\|\mathcal{A}\|$ for scale invariance: $\mathcal{A}_{norm} = \frac{1}{\|\mathcal{A}\|} \mathcal{A}$, $\mathcal{A}_{pinv} = \mathcal{A}_{pinv} \cdot \|\mathcal{A}\|$ [S1, unofficial].
2.  **Iteration Loop ($i = 0$ to $n_{iter}-1$):**
    *   **Compute Gradients:**
        *   Forward projection: $y_{pred} = \mathcal{A}(x)$ [S1, unofficial].
        *   Data Gradient:
            *   For linear models: $g_{data} = \mathcal{A}^*(y - y_{pred})$ [S1, unofficial].
            *   For nonlinear models (e.g., CT with attenuation): $y_{pred} = \exp(-\mu_{water} \cdot \mathcal{A}(x))$, then $g_{data} = \mu_{water} \cdot \mathcal{A}^*(y - y_{pred})$ [S1, unofficial].
        *   Regularization Gradient (optional, see variants): $g_{reg} = \mathcal{R}^*(x)$, where $\mathcal{R}^*$ is the adjoint of the regularizer (e.g., Laplacian: $(\partial_0^* \partial_0 + \partial_1^* \partial_1)(x)$) [S1, unofficial].
    *   **Concatenate Inputs:**
        *   Form input tensor $u$ by concatenating along the channel axis:
            *   No Regularizer: $u = \text{concat}(x, g_{data}, s)$ [S1, unofficial].
            *   With Regularizer: $u = \text{concat}(x, g_{data}, g_{reg}, s)$ [S1, unofficial].
    *   **Neural Network Update:**
        *   Layer 1: $h_1 = \text{ReLU}(W_1 * u + b_1)$ [S1, unofficial].
        *   Layer 2: $h_2 = \text{ReLU}(W_2 * h_1 + b_2)$ [S1, unofficial].
        *   Layer 3 (Linear): $h_3 = W_3 * h_2 + b_3$ [S1, unofficial].
    *   **Split Output:**
        *   Step size: $dx = h_3[:, :, :, 0:1]$ [S1, unofficial].
        *   New Memory: $s_{new} = \text{ReLU}(h_3[:, :, :, 1:])$ [S1, unofficial].
    *   **Update State:**
        *   $s \leftarrow s_{new}$ [S1, unofficial].
        *   $x \leftarrow x + dx$ [S1, unofficial].
3.  **Training (if learning weights):**
    *   Loss: Mean Squared Error between final $x$ and ground truth $x_{true}$: $\text{Loss} = \frac{1}{N} \sum_{b=1}^{N} \sum_{i,j} (x_{b,i,j} - x_{true,b,i,j})^2$ [S1, unofficial].
    *   Optimizer: RMSProp with inverse time decay [S1, unofficial].
        *   Starter learning rate: $10^{-3}$ [S1, unofficial].
        *   Decay rate: $1.0$ [S1, unofficial].
        *   Decay steps: $500$ [S1, unofficial].
        *   Staircase: True [S1, unofficial].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def partially_learned_gradient_descent(x_0, y, operator_adjoint, operator_forward, 
                                       n_iter, n_memory, use_regularizer=False, 
                                       mu_water=0.02, operator_norm=1.0,
                                       w1=None, b1=None, w2=None, b2=None, w3=None, b3=None):
    """
    Implements Partially Learned Gradient Descent.
    
    Args:
        x_0: Initial estimate tensor [batch, H, W, 1]
        y: Measurement tensor [batch, H_sin, W_sin, 1]
        operator_adjoint: Function computing A^*(residual)
        operator_forward: Function computing A(x)
        n_iter: Number of unrolled iterations
        n_memory: Number of memory channels
        use_regularizer: Boolean, whether to include regularization gradient in input
        mu_water: Attenuation coefficient for nonlinear forward model
        operator_norm: Norm of the forward operator for scaling
        w1, b1, w2, b2, w3, b3: Convolutional weights and biases
    Returns:
        x: Final reconstruction tensor
    """
    import numpy as np
    
    # Initialize memory state s to zeros
    batch_size = x_0.shape[0]
    H, W = x_0.shape[1], x_0.shape[2]
    s = np.zeros((batch_size, H, W, n_memory))
    
    x = x_0.copy()
    
    # Ensure weights are provided
    if w1 is None:
        raise ValueError("Weights must be provided or initialized.")

    for i in range(n_iter):
        # 1. Compute forward projection
        y_pred = operator_forward(x) / operator_norm
        
        # 2. Compute data gradient
        # Nonlinear model: y_pred = exp(-mu_water * A(x))
        # Residual: y - y_pred
        # Gradient: mu_water * A^*(y - y_pred)
        residual = y - np.exp(-mu_water * y_pred)
        gradx = operator_adjoint(residual) * mu_water
        
        # 3. Compute regularization gradient (if enabled)
        if use_regularizer:
            gradreg = compute_laplacian_adjoint(x)
            update_input = np.concatenate([x, gradx, gradreg, s], axis=-1)
        else:
            update_input = np.concatenate([x, gradx, s], axis=-1)
            
        # 4. Neural Network Layers
        h1 = np.maximum(0, conv2d(update_input, w1) + b1)
        h2 = np.maximum(0, conv2d(h1, w2) + b2)
        h3 = conv2d(h2, w3) + b3
        
        # 5. Split output
        dx = h3[:, :, :, 0:1]
        s_new = np.maximum(0, h3[:, :, :, 1:])
        
        # 6. Update states
        s = s_new
        x = x + dx
        
    return x

def conv2d(x, W, stride=(1, 1)):
    """Standard 2D convolution with SAME padding."""
    # x: [B, H, W, C_in], W: [kh, kw, C_in, C_out]
    # Returns: [B, H, W, C_out]
    B, H, W, C_in = x.shape
    kh, kw, _, C_out = W.shape
    out = np.zeros((B, H, W, C_out))
    for b in range(B):
        for c_out in range(C_out):
            for h in range(H):
                for w in range(W):
                    h_start = max(0, h - kh // 2)
                    h_end = min(H, h + (kh + 1) // 2)
                    w_start = max(0, w - kw // 2)
                    w_end = min(W, w + (kw + 1) // 2)
                    patch = x[b, h_start:h_end, w_start:w_end, :]
                    out[b, h, w, c_out] = np.sum(patch * W[:, :, :, c_out])
    return out

def compute_laplacian_adjoint(x):
    """
    Computes the adjoint of the Laplacian operator:
    (partial0.adjoint * partial0 + partial1.adjoint * partial1)(x)
    Implemented as a discrete convolution kernel.
    """
    # Kernel for 2D Laplacian adjoint (which is the Laplacian itself for symmetric operators)
    # Standard 5-point stencil:
    #  0 -1  0
    # -1  4 -1
    #  0 -1  0
    kernel = np.array([[[[0, -1, 0],
                        [-1, 4, -1],
                        [0, -1, 0]]]])
    # Expand kernel for batch/channel dimensions if necessary
    # Assuming x is [B, H, W, 1], kernel should be [3, 3, 1, 1]
    kernel_expanded = np.expand_dims(kernel, axis=-1)
    return conv2d(x, kernel_expanded)
```

**Parameters:**
*   $n_{memory}$: Number of memory channels. Typical value: 5 [S1, unofficial].
*   $n_{iter}$: Number of unrolled iterations. Typical value: 10 [S1, unofficial].
*   $n_{angles}$: Number of projection angles. Typical value: 30 [S1, unofficial].
*   $\mu_{water}$: Attenuation coefficient. Typical value: 0.02 [S1, unofficial].
*   $photons_{per\_pixel}$: For Poisson noise simulation. Typical value: 10,000 [S1, unofficial].
*   $\epsilon$: Small constant for log/noise. Value: $1/photons_{per\_pixel}$ [S1, unofficial].
*   Noise level: 5% of mean absolute data value [S1, unofficial].

**Pitfalls:**
*   **Nonlinear Saturation:** The exponential function in the nonlinear forward model can saturate if the input is not scaled correctly or if $\mu_{water}$ is incorrect. Ensure proper scaling of the forward projection before applying `exp` [S1, unofficial].
*   **Momentum Sign:** The memory state $s$ is updated using ReLU, ensuring it remains non-negative. This prevents oscillatory behavior from negative momentum values [S1, unofficial].
*   **Operator Norm:** Failure to normalize the operator by its norm can lead to scale-dependent convergence issues. The operator and pseudoinverse must be scaled by $\|\mathcal{A}\|$ and $1/\|\mathcal{A}\|$ respectively [S1, unofficial].

## Subtopics

### Partially Learned Gradient Descent (With Regularizer)
Includes a Tikhonov-like regularization term (adjoint of partial derivatives) in the network input concatenation [S1, unofficial]. Input channels are $n_{memory} + 3$ (estimate, data gradient, regularization gradient, memory) [S1, unofficial]. Weights $W_1$ shape: `[3, 3, n_memory + 3, 32]` [S1, unofficial].

### Sample Reuse
A technique to reuse samples from the previous gradient-descent iteration at the current iteration by merging reservoirs from current and previous iterations, treating different iterations as different frames [S2]. Handles sign changes by reusing both positive and negative samples for both reservoirs [S2]. $M_{cap}$ is typically 32 for base color, 16 otherwise [S2].

### Offset Refinement Recursion
A recursive inference step to refine noisy primitive positions by updating offsets $\Delta \mu_i$ based on previous estimation $\Delta \mu_{prev}$ [S3]. Update rule: $\Delta \mu_i^k = \text{Tanh}(\mathcal{D}_{pos}(\mathbf{f}_i)) \cdot v_{size}$, where $\mathbf{f}_i = \mathcal{F}(\mu_i^{init} + \Delta \mu_i^{k-1})$ [S3]. Stabilizes at a stationary point [S3]. On Waymo dataset, first inference increases PSNR by ~0.46 dB, subsequent updates maintain stable PSNR of 23.78 dB [S3].

## Operating characteristics
*   **Time Complexity:** Linear in the number of unrolled iterations $n_{iter}$ [S1, unofficial]. Each iteration involves one forward projection, one adjoint projection, and a small convolutional network pass [S1, unofficial].
*   **Memory Complexity:** Proportional to $n_{memory}$ channels for the state tensor $s$, plus the memory for the convolutional weights [S1, unofficial].
*   **Performance:** Outperforms standard gradient descent and FBP in terms of reconstruction quality (PSNR/SSIM) on medical imaging and phantom data [S1, unofficial].
*   **Strengths:** Retains physical model interpretability; learns adaptive step sizes and momentum; robust to noise [S1, unofficial].
*   **Limitations:** Requires training data to learn the network weights; inference speed depends on $n_{iter}$ and network depth; sensitive to operator norm scaling [S1, unofficial].
*   **When to use:** When a physics-based model is available but standard optimization converges slowly or gets stuck in local minima, and end-to-end learning is undesirable due to lack of interpretability [S1, unofficial].

## Background
Partially Learned Gradient Descent originates from the field of learned iterative reconstruction for medical imaging and tomography [S1, unofficial]. It was developed to bridge the gap between model-based iterative reconstruction (which is interpretable but slow/hard to tune) and end-to-end deep learning (which is fast but lacks physical guarantees) [S1, unofficial]. The method unrolls the optimization steps of gradient descent into a neural network, allowing the network to learn the update rule (step size and momentum) while keeping the data fidelity term fixed [S1, unofficial]. It is implemented in the `learned_gradient_tomography` repository [S1, unofficial].

## Sources
[S1] adler-j/learned_gradient_tomography, https://github.com/adler-j/learned_gradient_tomography, fetched 2026-10-27 (unofficial)
[S2] Parameter-space ReSTIR for Differentiable and Inverse Rendering, https://doi.org/10.1145/3588432.3591512, fetched 2026-10-27
[S3] EVolSplat: Efficient Volume-based Gaussian Splatting for Urban View Synthesis, https://doi.org/10.1109/CVPR52734.2025.01054, fetched 2026-11-03
