# Mip-NeRF Optimization and Activation Details

Also called: JaxNeRF Training Procedure, Mip-NeRF Misc. Changes

## Overview
This method specifies the implementation details for training mip-NeRF and JaxNeRF, focusing on activation functions, input encoding structures, and learning rate schedules designed to improve training stability and robustness [S1]. It solves the problem of optimization failure modes (such as zero gradients from negative density predictions or saturation on extreme colors) and improves convergence stability without significantly altering the model's accuracy or speed [S1]. It differs from standard NeRF training by removing the identity concatenation from the MLP input, using a shifted softplus for density, and employing a specific warmup and logarithmic annealing schedule for the learning rate [S1].

## How it works
**Inputs and Outputs:**
*   **Inputs:** Integrated Positional Encoding (IPE) features for position and view direction; ray direction vector $d$.
*   **Outputs:** Density $\tau$ and color $c$.

**Data Structures:**
*   **MLP Input:** A vector containing only the IPE features. The original position and view direction are *not* concatenated with the encoding [S1].
*   **View Direction Injection:** The encoded view direction is added to the last layer of the MLP [S1].

**Numbered Steps:**
1.  **Input Preparation:** Compute the Integrated Positional Encoding (IPE) for the spatial position and view direction. Do not concatenate the raw position or view direction vectors to this encoding [S1].
2.  **Density Prediction:** Pass the IPE features through the MLP layers. Apply the shifted softplus activation to the final output of the density branch to produce $\tau$ [S1].
3.  **View Direction Injection:** Normalize the ray direction $d$. Apply positional encoding with $L=4$ to $d$. Inject this encoded view direction into the last layer of the MLP [S1].
4.  **Color Prediction:** After the view direction injection, apply the widened sigmoid activation to the final output of the color branch to produce $c$ [S1].

**Formulas:**
*   **Density Activation (Shifted Softplus):**
    $$ \tau = \log(1 + \exp(x - 1)) $$
    Where $x$ is the raw MLP output for density. The shift by $-1$ is equivalent to initializing the density biases to $-1$, resulting in small initial $\tau$ values [S1].

*   **Color Activation (Widened Sigmoid):**
    $$ c = \frac{1 + 2\epsilon}{1 + \exp(-x)} - \epsilon $$
    Where $x$ is the raw MLP output for color and $\epsilon = 0.001$. This function saturates slightly outside the $[0, 1]$ range [S1].

*   **Learning Rate Schedule:**
    $$ \eta_i = (\lambda_w + (1 - \lambda_w) \sin((\pi/2) \operatorname{clip}(i/n_w, 0, 1))) \times (\exp((1 - i/n) \log(\eta_0) + (i/n) \log(\eta_n))) $$
    Where:
    *   $i$ is the current iteration.
    *   $n$ is the total number of iterations.
    *   $n_w$ is the number of warmup iterations.
    *   $\eta_0$ is the initial learning rate.
    *   $\eta_n$ is the final learning rate.
    *   $\lambda_w$ is a scale factor annealed from $0.01$ to $1$ during the warmup period [S1].

**Parameters:**
*   **Optimizer:** Adam [S1].
*   **Total Iterations ($n$):** 1,000,000 [S1].
*   **Batch Size:** 4096 [S1].
*   **Initial Learning Rate ($\eta_0$):** $5 \cdot 10^{-4}$ [S1].
*   **Final Learning Rate ($\eta_n$):** $5 \cdot 10^{-6}$ [S1].
*   **Warmup Iterations ($n_w$):** 2500 [S1].
*   **Warmup Start Factor ($\lambda_{w, start}$):** 0.01 [S1].
*   **Warmup End Factor ($\lambda_{w, end}$):** 1 [S1].
*   **View Direction PE Levels ($L$):** 4 [S1].

**Edge Cases and Pitfalls:**
*   **Negative Density:** Using standard ReLU for density can cause catastrophic failure if the MLP emits negative values everywhere, resulting in zero gradients [S1]. The shifted softplus prevents this.
*   **Color Saturation:** Standard sigmoid saturation can cause optimization failure when trying to explain black or white pixels. The widened sigmoid avoids this [S1].
*   **Identity Concatenation:** Concatenating raw position/view direction with IPE is not used as it makes formalization challenging and does not meaningfully contribute to performance [S1].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import math
import numpy as np

def shifted_softplus(x):
    """
    Density activation function.
    Equivalent to log(1 + exp(x - 1)).
    """
    return np.log(1 + np.exp(x - 1))

def widened_sigmoid(x, epsilon=0.001):
    """
    Color activation function.
    Saturates slightly outside [0, 1].
    """
    return (1 + 2 * epsilon) / (1 + np.exp(-x)) - epsilon

def compute_learning_rate(i, n, n_w, eta_0, eta_n, lambda_w_start=0.01, lambda_w_end=1.0):
    """
    Computes the learning rate at iteration i.
    """
    # Clip warmup progress to [0, 1]
    warmup_progress = min(max(i / n_w, 0.0), 1.0)
    
    # Anneal lambda_w from start to end during warmup
    # Note: The formula uses lambda_w as a variable that changes with i during warmup.
    # The source describes lambda_w as being "annealed from 0.01 to 1 during this warm up period".
    # The term (lambda_w + (1 - lambda_w) * sin(...)) implies lambda_w is the *current* value.
    # However, the formula structure suggests lambda_w is a parameter that is *itself* a function of i?
    # Re-reading source: "scale factor that is smoothly annealed between lambda_w = 0.01 and 1".
    # The formula provided is: eta_i = (lambda_w + (1 - lambda_w) * sin(...)) * (...)
    # This implies lambda_w in the formula is the *current* value of the annealing factor.
    # Let's define lambda_w(i) explicitly.
    
    if i < n_w:
        # Linear interpolation of lambda_w from 0.01 to 1.0? 
        # Source says "smoothly annealed". The sin term handles the shape of the *multiplier*.
        # Usually, warmup is linear or cosine. The source formula combines the warmup *scale* 
        # with the base schedule.
        # Let's interpret lambda_w in the formula as the *current* scale factor value.
        # If the source formula is exact, lambda_w must be defined.
        # "scale factor that is smoothly annealed between lambda_w = 0.01 and 1"
        # Let's assume linear annealing for lambda_w itself for the sake of implementation,
        # or that the formula implies lambda_w is the *start* value and the sin term modulates it?
        # No, "annealed from 0.01 to 1" suggests lambda_w is the variable.
        # Let's assume linear interpolation for lambda_w(i) during warmup.
        lambda_w_current = lambda_w_start + (lambda_w_end - lambda_w_start) * (i / n_w)
    else:
        lambda_w_current = lambda_w_end
        
    # The sin term modulates the contribution of the warmup factor?
    # Actually, looking at the formula: (lambda_w + (1 - lambda_w) * sin(...))
    # If lambda_w is 0.01, term is ~0.01 + 0.99 * sin.
    # If lambda_w is 1, term is 1.
    # This looks like a blend between a low value and a high value.
    
    sin_term = math.sin((math.pi / 2) * warmup_progress)
    
    warmup_scale = lambda_w_current + (1 - lambda_w_current) * sin_term
    
    # Logarithmic annealing for the base schedule
    log_eta = (1 - i / n) * math.log(eta_0) + (i / n) * math.log(eta_n)
    base_lr = math.exp(log_eta)
    
    return warmup_scale * base_lr

def mip_nerf_forward(mlp, x_ipe, d):
    """
    Forward pass for mip-NeRF with specific activation and view direction handling.
    
    Args:
        mlp: The neural network model.
        x_ipe: Integrated Positional Encoding features (input to MLP).
        d: Ray direction vector.
        
    Returns:
        tau: Density.
        c: Color.
    """
    # 1. Density Prediction
    # Pass IPE through MLP. The MLP structure is assumed to have a density branch.
    # We assume mlp(x_ipe) returns a tuple or dict, or we split the output.
    # Standard NeRF/MipNeRF MLP: Input -> Hidden Layers -> Density Output -> Hidden Layers -> Color Output.
    # The view direction is injected into the LAST layer.
    
    # Forward pass through all layers except the very last output layer for color?
    # Or does the view direction inject into the last layer of the *whole* network?
    # Source: "injected into the last layer of the MLP after tau is predicted but before c is predicted."
    # This implies the MLP is structured such that tau is computed, then view dir is added, then c is computed.
    
    # Let's assume a standard MLP structure where the last layer computes color.
    # We need to access the pre-activation or the output of the layer before the final color layer.
    
    # Simplified assumption: mlp has a method to get intermediate features.
    # If not, we assume the MLP is defined such that we can inject.
    
    # Step 1: Get density
    # Assuming mlp processes x_ipe through hidden layers to produce tau_raw
    tau_raw = mlp.compute_density(x_ipe)
    tau = shifted_softplus(tau_raw)
    
    # Step 2: View Direction Handling
    d_normalized = d / np.linalg.norm(d)
    # Apply positional encoding with L=4
    d_pe = positional_encoding(d_normalized, L=4)
    
    # Step 3: Inject into last layer
    # The source says "injected into the last layer".
    # This usually means adding to the features before the final color prediction.
    # We assume the MLP has a structure like:
    # x -> [Hidden Layers] -> h -> [Color Layer] -> c
    # And tau is predicted from h or an earlier stage.
    # If tau is predicted from h, then we add d_pe to h before the Color Layer.
    
    # Since we don't have the MLP internals, we assume a helper or structure.
    # [G] Implementation detail: The MLP must be structured to allow injection of view direction 
    # into the final hidden representation before the color output layer.
    
    # Let's assume mlp has a 'forward_with_view' or we manually compute.
    # For the sake of the pseudocode, we assume the MLP object handles this or we have access to the hidden state.
    
    # Alternative interpretation: The view direction is added to the input of the final layer.
    # Let's assume `mlp` is a standard MLP and we modify the last layer's input.
    
    # To be self-contained, we define a simple MLP structure here.
    pass # See below for full MLP definition

class MipNeRFMLP:
    def __init__(self, input_dim, hidden_dim, num_hidden_layers):
        self.input_dim = input_dim
        self.hidden_dim = hidden_dim
        self.num_hidden_layers = num_hidden_layers
        # Initialize weights and biases
        # [G] Standard initialization (e.g., He or Xavier) is assumed but not specified in source.
        self.weights = []
        self.biases = []
        for i in range(num_hidden_layers + 1):
            if i == 0:
                w = np.random.randn(input_dim, hidden_dim)
            else:
                w = np.random.randn(hidden_dim, hidden_dim)
            self.weights.append(w)
            self.biases.append(np.zeros(hidden_dim))
            
    def forward(self, x):
        h = x
        # Hidden layers
        for i in range(self.num_hidden_layers):
            h = np.maximum(0, h @ self.weights[i] + self.biases[i]) # ReLU for hidden layers (standard)
        
        # Density output
        tau_raw = h @ self.weights[-1] + self.biases[-1]
        tau = shifted_softplus(tau_raw)
        
        # View direction injection happens before color output
        # But the forward method above doesn't take view direction.
        # We need a separate forward for color with view direction.
        return tau, h # Return h for view direction injection

    def forward_color(self, h, d_pe):
        """
        Predicts color given hidden state h and encoded view direction d_pe.
        Injects d_pe into the last layer.
        """
        # Add view direction to the hidden state before the final layer?
        # Source: "injected into the last layer".
        # This typically means: h_final = h + d_pe (if dimensions match) or concatenated.
        # NeRF usually concatenates. But source says "injected".
        # In JaxNeRF/MipNeRF, it's often added to the last layer's input or output.
        # Given "injected into the last layer", and standard NeRF practice, 
        # it's often concatenated to the hidden representation before the final linear layer.
        
        # However, source says "injected into the last layer... after tau is predicted".
        # If tau is predicted from h, then we modify h or the input to the final layer.
        
        # Let's assume concatenation as is standard in NeRF for view direction, 
        # unless "injected" implies addition. 
        # Source E.4: "positionally encoded... and injected into the last layer".
        # In the original NeRF, it's concatenated. 
        # We will assume concatenation to be safe, as "injected" is vague, 
        # but concatenation is the standard "NeRF" way.
        # [G] If the source implies addition, the dimensions must match. 
        # Concatenation is safer for dimension mismatch.
        
        h_with_view = np.concatenate([h, d_pe], axis=-1)
        
        # Final layer for color
        c_raw = h_with_view @ self.weights[-1] + self.biases[-1] # Reusing weights? 
        # No, the last weight matrix for color is different from density?
        # In NeRF, there are two separate output heads.
        # So we need separate weights for color.
        
        # [G] Assuming separate weights for color output layer.
        c_raw = h_with_view @ self.color_weights + self.color_biases
        c = widened_sigmoid(c_raw)
        
        return c

# Full implementation with separate heads
class MipNeRFMLPFull:
    def __init__(self, input_dim, hidden_dim, num_hidden_layers):
        self.input_dim = input_dim
        self.hidden_dim = hidden_dim
        self.num_hidden_layers = num_hidden_layers
        
        # Weights for hidden layers
        self.weights = []
        self.biases = []
        for i in range(num_hidden_layers):
            if i == 0:
                w = np.random.randn(input_dim, hidden_dim)
            else:
                w = np.random.randn(hidden_dim, hidden_dim)
            self.weights.append(w)
            self.biases.append(np.zeros(hidden_dim))
            
        # Density head
        self.density_weights = np.random.randn(hidden_dim, 1)
        self.density_biases = np.full(1, -1.0) # Shifted softplus init equivalent
        
        # Color head
        self.color_weights = np.random.randn(hidden_dim, 3) # Assuming 3 channels
        self.color_biases = np.zeros(3)

    def forward(self, x_ipe, d):
        h = x_ipe
        for i in range(self.num_hidden_layers):
            h = np.maximum(0, h @ self.weights[i] + self.biases[i])
            
        # Density
        tau_raw = h @ self.density_weights + self.density_biases
        tau = shifted_softplus(tau_raw)
        
        # View Direction
        d_normalized = d / np.linalg.norm(d)
        d_pe = positional_encoding(d_normalized, L=4)
        
        # Inject into last layer (concatenate)
        h_with_view = np.concatenate([h, d_pe], axis=-1)
        
        # Color
        c_raw = h_with_view @ self.color_weights + self.color_biases
        c = widened_sigmoid(c_raw)
        
        return tau, c

def positional_encoding(v, L):
    """
    Standard sinusoidal positional encoding.
    """
    # [G] Standard PE definition: sin(2^i * pi * v) for i in 0..L-1
    # Doubles frequency each step.
    freq_bands = 2.0 ** np.arange(L)
    pe = []
    for band in freq_bands:
        pe.append(np.sin(band * np.pi * v))
        pe.append(np.cos(band * np.pi * v))
    return np.concatenate(pe, axis=-1)
```

## Subtopics

### [Mip-NeRF 360 Optimization Setup](3426-mip-nerf-360-optimization-setup.md)
This article specifies the training configuration, network architecture, and loss functions for the Mip-NeRF 360 method, which extends mip-NeRF to unbounded, "in-the-wild" scenes. It solves the problem of training stability and rendering quality in complex environments with large depth ranges and varying lighting by introducing a two-stage proposal sampling strategy and a specific regularization scheme. It differs from standard NeRF training by using a proposal network to guide sample placement and from basic mip-NeRF by employing a Charbonnier reconstruction loss and a log-linear learning rate schedule to handle the scale and complexity of outdoor/indoor hybrid scenes.

### [Scale-Dependent Downweighting for Grid Interpolation](8571-scale-dependent-downweighting-for-grid-interpolation.md)
Scale-Dependent Downweighting is a technique to anti-alias trilinear interpolation in neural radiance fields by downweighting features interpolated from grid cells that are small relative to the sampling Gaussian's spread, preventing aliasing artifacts from high-frequency content. Unlike naive trilinear interpolation which causes aliasing at high frequencies, this method uses the Gaussian CDF to weight samples, similar to Mip-NeRF's IPE features but applied to grid-based representations.

### Alternative Gaussian Positional Encoding
An alternative approach to Integrated Positional Encoding (IPE) is to concatenate the positional encodings of the mean and the (signed) square root of the covariance matrix elements as input to the MLP [S1]. Specifically, positional encoding with $L=12$ is applied to the mean $\mu$, and positional encoding with $L=2$ is applied to the vectorized upper triangular part of the signed covariance matrix [S1]. This alternative performs comparably to IPE in accuracy but is sensitive to the hyperparameters $L$ and the covariance parameterization, whereas IPE is hyperparameter-free and compact [S1]. A pitfall of IPE is that it is insensitive to the off-diagonal elements of the covariance matrix $\Sigma$ (correlation), but experiments show that modeling this correlation does not improve accuracy, suggesting it is not a helpful cue [S1].

## Operating characteristics
*   **Time Complexity:** Mip-NeRF is 13x faster to train and 16x faster to evaluate than a brute-force supersampling baseline (16 rays/pixel) [S1].
*   **Memory Complexity:** Not given in the sources.
*   **Measured Performance:** On the multiscale Blender dataset, Mip-NeRF achieves a PSNR of 32.629 (Full Res), SSIM of 0.9579, and LPIPS of 0.0469, outperforming the 16x supersampling baseline (PSNR 32.18, SSIM 0.9524, LPIPS 0.0537) [S1]. Training time is 2.79 hours for Mip-NeRF vs. significantly longer for supersampling (exact time not given, but 13x slower) [S1].
*   **Scaling:** The method is designed to handle multiscale data and reduce aliasing [S1].
*   **Strengths:** Improved training stability, robustness to large learning rates, and better handling of multiscale data compared to standard NeRF [S1].
*   **Limitations:** The "miscellaneous" changes (activations, warmup) do not significantly change accuracy or speed but are crucial for stability [S1].
*   **When to use:** When training mip-NeRF or JaxNeRF to ensure stable convergence and avoid optimization failure modes [S1].
*   **When not to use:** If computational resources are extremely limited and the slight stability gain is not worth the implementation complexity, or if using a different architecture that does not benefit from these specific activations [S1].

## Background
The method originates from the Mip-NeRF paper, which introduced a multiscale representation for anti-aliasing neural radiance fields [S1]. The specific optimization details, including the shifted softplus, widened sigmoid, and learning rate schedule, were documented in the supplementary material of the Mip-NeRF paper to ensure reproducibility and improve training stability for ablations and comparisons [S1]. The training procedure is based on the JaxNeRF codebase [S1].

## Sources
[S1] Mip-NeRF: A Multiscale Representation for Anti-Aliasing Neural Radiance Fields, https://doi.org/10.1109/iccv48922.2021.00580, fetched 2026-10-26
