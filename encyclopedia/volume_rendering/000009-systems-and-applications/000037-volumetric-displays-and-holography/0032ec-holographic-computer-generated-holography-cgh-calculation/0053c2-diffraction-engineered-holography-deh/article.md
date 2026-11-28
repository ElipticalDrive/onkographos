# Diffraction-Engineered Holography (DEH)

Also called: optimized DEH

## Overview
Diffraction-Engineered Holography (DEH) is a hologram synthesis method that optimizes a single complex wave field to reconstruct sharply focused images at object-existing planes and blurred images at other focal planes, thereby providing real-world-like defocus blur and photorealistic reconstruction [S1]. It sets itself apart from conventional methods by explicitly optimizing diffracted patterns at out-of-focus planes to enhance defocus blur while maintaining high image quality at in-focus planes, unlike siblings such as [nCGH](7015-diffraction-specific-approach.md) which exhibit weak defocus blur [S1].

## How it works
**Inputs:**
*   $N$ target varifocal images $I_{d_n}$, where $n=1 \dots N$, rendered for specific propagation distances $d_n$ using a physically-based renderer to ensure occluded surfaces are correctly handled [S1].
*   An all-in-focus image $I_{AIF}$ of the scene [S1].
*   A depth map $D$ of the scene [S1].
*   A depth attention factor $\eta$ [S1].
*   Initial complex wave field at the hologram plane ($z=0$): $U_0(x,y) = A(x,y)e^{i\phi(x,y)}$, typically initialized with constant amplitude and random phase [G].

**Outputs:**
*   Optimized complex wave field $U_{DEH}(x,y) = A(x,y)e^{i\phi_{opt}(x,y)}$ at the hologram plane [S1].

**Data Structures:**
*   **Wave Field:** Complex array of size $H \times W$ (hologram resolution), stored as two real arrays: Amplitude $A$ and Phase $\phi$ [G].
*   **Varifocal Images:** Array of $N$ intensity images of size $H_{img} \times W_{img}$ corresponding to depths $d_n$ [G].
*   **Depth Map:** Array of size $H_{img} \times W_{img}$ containing depth values $D(x,y)$ [G].

**Parameters:**
*   $N$: Number of varifocal planes. The benchmark dataset uses 21 planes [S1].
*   $\eta$: Depth attention factor. Value not explicitly given in sources [G].
*   Iterations: Optimization steps. Typical value: 500 [S1].
*   Wavelength $\lambda$: Required for wavenumber $k = 2\pi/\lambda$ [G].

**Steps:**

1.  **Render Varifocal Images:** Generate target intensity images $I_{d_n}$ for each propagation distance $d_n$ using a physically-based renderer. The camera focal distance is changed to match $d_n$ to capture accurate blur circles and occlusions [S1].

2.  **Initialize Wave Field:** Initialize the complex wave field $U_0(x,y)$ at $z=0$ [G].

3.  **Optimization Loop:** Repeat for $iter = 1$ to $MaxIter$:
    a. **Propagation:** For each target plane $n=1 \dots N$:
        i. Calculate the propagated wave field $Prop_{d_n}$ at distance $d_n$ using the Angular Spectrum Method (ASM) [S1]:
        $$ Prop_{d_n}(U_0) = \mathcal{F}^{-1} \left[ \mathcal{F}[U_0(x,y)] \cdot e^{i k_z d_n} \right] $$
        where $\mathcal{F}$ and $\mathcal{F}^{-1}$ are the Fourier and inverse Fourier transform operators, $k = 2\pi/\lambda$, and $k_z = \sqrt{k^2 - k_x^2 - k_y^2}$ is the axial wavenumber, with $k_x, k_y$ being the angular wavenumbers in the lateral directions [S1].
        ii. Calculate the propagated intensity: $I_{prop,n} = |Prop_{d_n}(U_0)|^2$ [S1].

    b. **Loss Calculation:** Compute the total loss $L$ [S1]:
        $$ L = \sum_{n=1}^{N} \text{MSE}(I_{prop,n}, I_{d_n}) + \lambda_{AIF} \cdot \text{MSE}(I_{prop,closest}, I_{AIF\_weighted}) $$
        The first term is the MSE between propagated intensity and varifocal images [S1]. The second term is the MSE between propagated intensity and a depth-weighted all-in-focus image, weighted by $\eta$ and $D$ [S1]. $I_{AIF\_weighted}(x,y) = I_{AIF}(x,y) \cdot w(D(x,y))$ where $w$ is a depth-dependent weight function emphasizing planes close to object depths [G]. The index $closest$ refers to the plane $d_n$ closest to the object depth for each pixel [G].

    c. **Gradient Descent:** Update the phase $\phi(x,y)$ of the wave field using gradient descent to minimize $L$ [S1].
        $$ \phi_{new}(x,y) = \phi_{old}(x,y) - \alpha \frac{\partial L}{\partial \phi(x,y)} $$
        where $\alpha$ is the learning rate [G].

    d. **Convergence Check:** If the change in the wave field (e.g., phase difference) is negligible, stop [S1].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import numpy as np
from scipy.fft import fft2, ifft2

def angular_spectrum_propagation(U0, dz, wavelength, pixel_size):
    """
    Propagates a complex wave field U0 by distance dz using Angular Spectrum Method.
    """
    k = 2 * np.pi / wavelength
    H, W = U0.shape
    fx = np.fft.fftfreq(W, d=pixel_size)
    fy = np.fft.fftfreq(H, d=pixel_size)
    Fx, Fy = np.meshgrid(fx, fy)
    kx = 2 * np.pi * Fx
    ky = 2 * np.pi * Fy
    kx2_plus_ky2 = kx**2 + ky**2
    # Handle evanescent waves by setting imaginary kz for exponential decay [G]
    kz = np.where(kx2_plus_ky2 <= k**2, np.sqrt(k**2 - kx2_plus_ky2), 1j * np.sqrt(kx2_plus_ky2 - k**2))
    kernel = np.exp(1j * kz * dz)
    U_freq = fft2(U0)
    U_prop_freq = U_freq * kernel
    return ifft2(U_prop_freq)

def compute_mse(img1, img2):
    return np.mean((img1 - img2)**2)

def deh_optimization(varifocal_images, depths, aif_image, depth_map, eta, wavelength, pixel_size, num_iterations=500, learning_rate=0.1):
    """
    Optimizes a hologram wave field using Diffraction-Engineered Holography.
    """
    H, W = varifocal_images[0].shape
    A = np.ones((H, W))
    phi = np.random.uniform(0, 2 * np.pi, (H, W))
    U = A * np.exp(1j * phi)

    for i in range(num_iterations):
        total_loss = 0.0
        
        # Term 1: MSE against varifocal images
        for n in range(len(varifocal_images)):
            d_n = depths[n]
            I_target = varifocal_images[n]
            U_prop = angular_spectrum_propagation(U, d_n, wavelength, pixel_size)
            I_prop = np.abs(U_prop)**2
            total_loss += compute_mse(I_prop, I_target)

        # Term 2: MSE against depth-weighted all-in-focus image
        # Source does not specify exact weighting function w(D) or closest-plane selection [G]
        # Assuming a simplified global comparison to a reference depth d_ref
        d_ref = np.median(depths)
        U_prop_aif = angular_spectrum_propagation(U, d_ref, wavelength, pixel_size)
        I_prop_aif = np.abs(U_prop_aif)**2
        I_aif_weighted = aif_image * (1.0 - depth_map / np.max(depth_map)) * eta
        total_loss += compute_mse(I_prop_aif, I_aif_weighted)

        # Gradient descent update
        # The exact gradient dL/dphi requires backpropagation through the ASM operators [G]
        # Sources do not provide the explicit analytical derivative [G]
        grad_L = np.zeros_like(phi) # Placeholder for backprop result
        phi = phi - learning_rate * grad_L

        if i % 100 == 0:
            print(f"Iteration {i}, Loss: {total_loss}")

    return A * np.exp(1j * phi)
```

## Subtopics

### [DEHNet](15065-dehnet.md)
DEHNet is a convolutional neural network designed to synthesize Diffraction-Engineered Hologram (DEH) wave fields directly from an all-in-focus color image and an all-in-focus depth map. It solves the computational bottleneck of traditional DEH generation, which requires multiple varifocal images and iterative optimization, by replacing the optimization procedure with a direct inference step. Unlike its sibling method nCGH, which also uses RGB-D inputs but lacks depth-aware defocus control, DEHNet produces wave fields that correctly reconstruct defocus blur and sharp focus while suppressing interference artifacts at object boundaries.

## Operating characteristics
**Time Complexity:** The optimization-based DEH is computationally expensive [S1]. It requires synthesizing multiple varifocal images and performing iterative optimization (e.g., 500 iterations) [S1]. Synthesis takes more than 1 minute for a single hologram on an NVIDIA V100 GPU [S1]. DEHNet achieves 62 Hz inference on FHD resolution images [S1].

**Memory Complexity:** Requires storing the complex wave field, varifocal images, and depth maps [G]. The optimization process requires memory for gradients and intermediate propagated fields [G].

**Performance:** General dataset: PSNR 26.1 dB, SSIM 0.88 [S1]. 512x512 dataset: PSNR 29.0 dB, SSIM 0.909 [S1]. FHD dataset: PSNR 33.4 dB, SSIM 0.955 [S1].

**Strengths:** Provides clear defocus blur even if the blur circle diameter is slightly smaller than the rendered image, enhancing depth perception [S1]. Maintains high image quality at in-focus planes [S1].

**Limitations:** High computational load for the optimization-based method makes it impractical for real-time applications without neural network acceleration [S1].

**When to use:** When photorealistic depth expression with accurate defocus blur is required and computational resources allow for offline synthesis or neural network inference [G].

**When not to use:** For real-time applications requiring rapid hologram generation without pre-trained models, or when hardware constraints prevent the use of large neural networks [G].

## Background
Diffraction-Engineered Holography (DEH) was introduced to address the limitations of conventional CGH algorithms that only optimize intensity at object-existing planes, failing to provide realistic defocus blur at other planes [S1]. It leverages the fact that phase variation steers light propagation without affecting the image seen by the eyes, allowing adaptive phase changes to enhance defocus blur [S1]. The method was demonstrated in a 2022 paper published in *Nature Communications* [S1].

## Sources
[S1] Diffraction-engineered holography: Beyond the depth representation limit of holographic displays, https://www.nature.com/articles/s41467-022-33728-5, fetched 2026-10-29
