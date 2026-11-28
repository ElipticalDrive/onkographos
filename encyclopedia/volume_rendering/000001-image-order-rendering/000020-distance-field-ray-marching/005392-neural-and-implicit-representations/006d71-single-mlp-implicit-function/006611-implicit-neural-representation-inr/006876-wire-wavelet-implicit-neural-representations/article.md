# WIRE (Wavelet Implicit Neural Representations)

Also called: Gabor Wavelet INR, Wavelet Implicit Neural Representations

## Overview
WIRE is an Implicit Neural Representation (INR) that uses a continuous complex Gabor wavelet as the activation nonlinearity in its Multi-Layer Perceptron (MLP) to achieve high accuracy, fast training, and robustness to noise by leveraging optimal space-frequency compactness [S1]. It differs from [SIREN](19545-implicit-neural-representation-inr.md) by adding a Gaussian window to provide spatial compactness, and from pure Gaussian activations by adding a complex exponential term to provide frequency compactness [S1]. This combination allows WIRE to decompose images as a linear combination of sparse images, providing high representational capacity for a given parameter count [S1].

## How it works
**Inputs and Outputs:**
*   **Input:** Coordinates $x \in \mathbb{R}^{D_i}$ (e.g., 2D image coordinates or 3D spatial coordinates).
*   **Output:** A real-valued scalar (e.g., pixel intensity, density, or color component).

**Data Structures:**
*   **Weights and Biases:** The MLP weights $W_m$ and biases $b_m$ are **complex-valued** to preserve phase relationships throughout the network [S1].
*   **Layer Outputs:** Intermediate activations $y_m$ are complex-valued [S1].
*   **Final Output:** The real part of the final network output is taken; the imaginary part is discarded [S1].

**Activation Function:**
The core nonlinearity is the Gabor wavelet $\sigma(x)$, defined as:
$$ \sigma(x) = \psi(x; \omega_0, s_0) = e^{j\omega_0 x} e^{-|s_0 x|^2} $$
where:
*   $\omega_0$ controls the frequency of the sinusoid.
*   $s_0$ controls the spread (width) of the Gaussian envelope.
*   $j$ is the imaginary unit.

An alternate form for problems where complex weights are infeasible uses the real (or imaginary) part of the complex Gabor wavelet:
$$ \psi(x; \omega_0, s_0) = \sin(\omega_0 x)e^{-(s_0 x)^2} $$
Note that setting $s_0 = 0$ results in the pure sine nonlinearity used in SIREN, and setting $\omega_0 = 0$ results in the Gaussian nonlinearity [S1].

**Multi-dimensional Localization (2D WIRE):**
For multidimensional localization, the output of each unit is computed by augmenting the Gabor wavelet with additional Gaussian windows:
$$ y_m = \psi(W_m y_{m-1} + b^{(1)}_m; \omega_0, s_0) \cdot e^{-\sum_{k=2}^{D_m} |s_0 (W_m y_{m-1} + b_m)|^2} $$
This ensures activations are compact along both spatial axes, unlike 1D WIRE which may blur features along one axis [S1].

**Parameters:**
*   **$\omega_0, s_0$:** Default hyperparameters for standard image representation are $\omega_0 = 20$ and $s_0 = 10$ [S1]. A Constant-Q embodiment sets $\omega_0 s_0 = Q$ for constant fractional bandwidth [S1].
*   **Learning Rate:** WIRE performs best with a high learning rate of $2 \times 10^{-2}$ [S1].
*   **Hidden Layers:** Fixed to 2 for standard image representation experiments [S1]. Using $\ge 3$ layers causes performance to converge to SIREN/Gaussian levels, increases computational cost, and often causes instability [S1].
*   **Hidden Features:** For NeRF applications, 128 features are used per layer (calculated as $182/\sqrt{2}$) to account for parameter doubling from complex weights [S1].

**Initialization:**
WIRE is robust to initialization and works with standard uniform initialization [S1]. However, it marginally benefits from SIREN-like initialization, which can yield up to 1dB higher accuracy [S1]:
*   First layer weights: $U(-1/N, 1/N)$
*   Subsequent layer weights: $U(-6/(\omega_0 N), 6/(\omega_0 N))$
where $N$ is the number of input features [S1].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import torch
import torch.nn as nn
import math

class GaborWaveletActivation(nn.Module):
    def __init__(self, omega_0=20.0, s_0=10.0, use_complex_weights=True):
        super().__init__()
        self.omega_0 = omega_0
        self.s_0 = s_0
        self.use_complex_weights = use_complex_weights
        # If using real-valued weights, we take the real part of the activation
        # If using complex weights, we keep the full complex activation

    def forward(self, x):
        # x shape: (..., D)
        # Compute the argument to the wavelet
        # Note: In the complex weight case, W and b are complex, so x is projected to complex space.
        # Here we assume x is real and W/b are complex, or x is complex.
        # The formula is psi(z) = exp(j * omega_0 * z) * exp(-|s_0 * z|^2)
        
        # For the real-part-only variant (no complex weights):
        # z = x (real)
        # psi = sin(omega_0 * x) * exp(-(s_0 * x)^2)
        
        # For the complex variant (standard WIRE):
        # z is complex. 
        # psi = exp(1j * omega_0 * z) * exp(-|s_0 * z|^2)
        
        if self.use_complex_weights:
            # z is complex
            term1 = torch.exp(1j * self.omega_0 * x)
            # |s_0 * z|^2 = (s_0 * z) * conj(s_0 * z) = s_0^2 * |z|^2
            term2 = torch.exp(- (self.s_0 * x).abs() ** 2)
            return term1 * term2
        else:
            # Real-valued variant
            return torch.sin(self.omega_0 * x) * torch.exp(-(self.s_0 * x) ** 2)

class WIREMLP(nn.Module):
    def __init__(self, input_dim, hidden_dim, output_dim, num_hidden_layers=2, 
                 omega_0=20.0, s_0=10.0, use_complex_weights=True):
        super().__init__()
        self.num_hidden_layers = num_hidden_layers
        self.activation = GaborWaveletActivation(omega_0, s_0, use_complex_weights)
        
        # Define layers
        # Note: If use_complex_weights is True, Linear layers must support complex weights
        # PyTorch native Linear does not support complex weights directly in older versions,
        # but modern PyTorch does. If not, one must implement complex linear manually.
        # Assuming modern PyTorch complex support or manual implementation.
        
        self.layers = nn.ModuleList()
        
        # First layer
        self.layers.append(nn.Linear(input_dim, hidden_dim, bias=True))
        
        # Hidden layers
        for _ in range(num_hidden_layers - 1):
            self.layers.append(nn.Linear(hidden_dim, hidden_dim, bias=True))
            
        # Output layer (Linear, no activation, produces complex output if complex weights used)
        self.output_layer = nn.Linear(hidden_dim, output_dim, bias=True)
        
        self._init_weights(omega_0, input_dim, hidden_dim)

    def _init_weights(self, omega_0, input_dim, hidden_dim):
        """
        SIREN-like initialization for better performance, though standard init is robust.
        """
        for layer in self.layers:
            if layer == self.layers[0]:
                # First layer: U(-1/N, 1/N)
                nn.init.uniform_(layer.weight, -1/math.sqrt(input_dim), 1/math.sqrt(input_dim))
                if layer.bias is not None:
                    nn.init.zeros_(layer.bias)
            else:
                # Subsequent layers: U(-6/(omega_0 * N), 6/(omega_0 * N))
                # N is the number of input features to this layer
                n = layer.weight.shape[1]
                limit = 6.0 / (omega_0 * math.sqrt(n))
                nn.init.uniform_(layer.weight, -limit, limit)
                if layer.bias is not None:
                    nn.init.zeros_(layer.bias)
                    
        # Output layer initialization (standard)
        n_out = self.output_layer.weight.shape[1]
        nn.init.uniform_(self.output_layer.weight, -1/math.sqrt(n_out), 1/math.sqrt(n_out))
        if self.output_layer.bias is not None:
            nn.init.zeros_(self.output_layer.bias)

    def forward(self, x):
        # x: (..., D_i)
        h = x
        for i, layer in enumerate(self.layers):
            h = layer(h)
            if i < self.num_hidden_layers: # Apply activation to hidden layers
                h = self.activation(h)
        
        # Output layer
        out = self.output_layer(h)
        
        # Take real part for real-valued signals
        if self.activation.use_complex_weights:
            return out.real
        else:
            return out

# Example usage for 2D WIRE (Multi-dimensional localization)
class WIRE2D(nn.Module):
    def __init__(self, input_dim, hidden_dim, output_dim, num_hidden_layers=2, 
                 omega_0=20.0, s_0=10.0):
        super().__init__()
        self.num_hidden_layers = num_hidden_layers
        self.omega_0 = omega_0
        self.s_0 = s_0
        self.layers = nn.ModuleList()
        
        # First layer
        self.layers.append(nn.Linear(input_dim, hidden_dim, bias=True))
        
        # Hidden layers
        for _ in range(num_hidden_layers - 1):
            self.layers.append(nn.Linear(hidden_dim, hidden_dim, bias=True))
            
        self.output_layer = nn.Linear(hidden_dim, output_dim, bias=True)
        
    def _init_weights(self, omega_0, input_dim, hidden_dim):
        for layer in self.layers:
            if layer == self.layers[0]:
                nn.init.uniform_(layer.weight, -1/math.sqrt(input_dim), 1/math.sqrt(input_dim))
            else:
                n = layer.weight.shape[1]
                limit = 6.0 / (omega_0 * math.sqrt(n))
                nn.init.uniform_(layer.weight, -limit, limit)

    def forward(self, x):
        h = x
        for i, layer in enumerate(self.layers):
            h = layer(h)
            if i < self.num_hidden_layers:
                # 2D WIRE Activation
                # psi(z) * exp(-sum_{k=2}^{D_m} |s_0 * z_k|^2)
                # Here z = h
                # We assume h has dimension D_m = hidden_dim
                # The first term is the Gabor wavelet applied to the whole vector? 
                # The paper formula: y_m = psi(W y_{m-1} + b^{(1)}; w0, s0) * exp(- sum_{k=2}^{D_m} |s_0 (W y_{m-1} + b_m)|^2)
                # This implies psi is applied to the first component or the norm? 
                # The text says "augmenting the Gabor wavelet with additional Gaussian windows".
                # Usually, this means the Gabor is applied to the first dimension or a specific projection, 
                # and Gaussians to the rest. However, the formula notation is slightly ambiguous.
                # A common interpretation for "multi-dimensional" in this context (Fig 11/12) 
                # is that the activation is separable or the Gaussian windows dampen the other dimensions.
                # Given the formula: psi(...) * exp(- sum_{k=2}^{D_m} ...), it suggests psi is applied to the first component (k=1) 
                # or the vector is treated such that the first term is the wavelet and others are Gaussian.
                # Let's implement the literal formula assuming the first component is the wavelet argument 
                # and the rest are Gaussian dampeners.
                
                # z = h
                z1 = z[:, 0:1] # First component
                z_rest = z[:, 1:] # Remaining components
                
                # Gabor on first component (assuming real weights for simplicity in this block, 
                # or complex if supported. Let's assume real for the 2D variant description 
                # as complex 2D is less standard in simple PyTorch without custom ops)
                # Using real part of complex Gabor for stability if complex weights not fully supported in this block
                gabor_val = torch.sin(self.omega_0 * z1) * torch.exp(-(self.s_0 * z1)**2)
                
                # Gaussian windows for rest
                gaussian_val = torch.exp(- (self.s_0 * z_rest)**2)
                gaussian_val = torch.prod(gaussian_val, dim=1, keepdim=True)
                
                h = gabor_val * gaussian_val
                
        out = self.output_layer(h)
        return out.real
```

## Operating characteristics
*   **Time and Memory Complexity:** WIRE converges an order of magnitude faster than SIREN and Gaussian INRs [S1]. However, using a large number of hidden layers ($\ge 3$) increases computational cost and often causes an unstable learning regime [S1].
*   **Measured Performance:**
    *   **Image Representation:** 43.2 dB PSNR on Kodak dataset (vs 42.4 dB SIREN, 40.0 dB Gauss) [S1].
    *   **Occupancy Volume:** 0.99 IOU on Thai statue mesh (vs 0.97 SIREN/Gauss) [S1].
    *   **Denoising:** 30.2 dB PSNR on DIV2K with 17.6 dB input noise (vs 28.1 dB DIP, 30.1 dB Gauss) [S1].
    *   **Super-Resolution:** 27.3 dB PSNR for 4x single image SR (vs 26.4 dB Bilinear, 25.6 dB SIREN) [S1].
    *   **NeRF:** +0.2 dB higher accuracy than SIREN with 25 training images; +0.4 dB with 100 images [S1].
*   **Strengths:** High accuracy, fast training, robustness to noise and undersampling, sparse second-layer outputs enabling high approximation capacity for high-frequency signals [S1].
*   **Limitations:** Requires complex weights for optimal performance (though real-part variants exist); performance degrades to SIREN/Gaussian levels with too many layers ($\ge 3$); 1D WIRE blurs features along one axis compared to 2D WIRE [S1].
*   **When to use:** Ideal for image representation, denoising, super-resolution, CT reconstruction, and NeRF training, especially when data is noisy or undersampled [S1].
*   **When not to use:** When computational resources are extremely limited and layer depth must be large, or when complex weight support is unavailable and real-valued approximations are insufficient [S1].

## Background
WIRE was introduced in the paper "WIRE: Wavelet Implicit Neural Representations" by Vishwanath et al., published on arXiv in January 2023 [S1]. It was developed to address the trade-off between high accuracy and robustness in INRs, leveraging the optimal space-frequency compactness of Gabor wavelets from harmonic analysis [S1]. It builds on the properties of SIREN (sinusoidal activations) and Gaussian activations, inheriting their favorable traits while mitigating their weaknesses (global ringing for SIREN, compact but large error at edges for Gauss) [S1].

## Sources
[S1] WIRE: Wavelet Implicit Neural Representations, http://arxiv.org/abs/2301.05187v1, fetched 2026-10-27
