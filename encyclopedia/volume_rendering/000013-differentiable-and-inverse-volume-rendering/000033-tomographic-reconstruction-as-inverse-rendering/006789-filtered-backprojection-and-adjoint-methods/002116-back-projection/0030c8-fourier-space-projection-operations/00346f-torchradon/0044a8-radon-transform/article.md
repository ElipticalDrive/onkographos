# Radon Transform
Also called: Projection intégrale continue, Sinogram Generation, Transformée de Radon

## Overview
The Radon transform maps a 2D spatial function to its line integrals along parallel rays at various angles, producing a sinogram [S1, S2, S6, unofficial]. It models the continuous tomographic forward acquisition process by integrating object contrast along straight projection paths [S2, S5, unofficial]. It is distinguished from other projection methods by its strict parallel-beam geometry and direct line-integral formulation [S1, S6, unofficial].

## How it works
**Inputs and Outputs:**
- **Input:** A 2D image $I$ (discrete pixel grid) and a set of projection angles $\theta$. For batch processing, the input is a tensor of shape $(N, C, H, W)$ where $N$ is batch size, $C$ channels, $H$ height, and $W$ width [S4, unofficial].
- **Output:** A sinogram tensor of shape $(N, C, D, A)$ where $D$ is the number of detector positions (radial bins) and $A$ is the number of angles [S4, unofficial].

**Data Structures:**
- **Image Grid:** Discrete indices $(i, j)$ corresponding to pixel centers. The image is modeled as a sum of discrete Dirac impulses [S5, unofficial]:
  $$ f(x,y) = \sum_{i=-\infty}^{\infty} \sum_{j=-\infty}^{\infty} I(i,j) \Delta(x - j) \Delta(y - i) $$
  where $\Delta(\cdot)$ is the discrete Dirac impulse [S5, unofficial].
- **Projection Line:** Defined by angle $\theta$ and radial distance $\rho$ (or $t$) from the origin. The line equation is $\rho = x \cos \theta + y \sin \theta$ [S1, S2, S5, unofficial].
- **Detector Positions:** Linearly spaced values $r_s$ covering the image extent, typically from $-1$ to $1$ normalized or in pixel units [S3, unofficial].

**Formulas:**
The continuous Radon transform is defined as [S1, S5, unofficial]:
$$ R_\theta(\rho) = \int_{-\infty}^{\infty} \int_{-\infty}^{\infty} f(x,y) \delta(\rho - x \cos \theta - y \sin \theta) \, dx \, dy $$
where $\delta(\cdot)$ is the Dirac delta function [S5, unofficial].

The discrete Radon transform $R_\theta(\rho)$ is the sum of pixel values $I(i,j)$ whose centers lie on the projection line [S5, unofficial]:
$$ R_\theta(\rho) = \sum_{i=0}^{H-1} \sum_{j=0}^{W-1} I(i,j) \cdot \mathbb{I}(\text{line } (\theta, \rho) \text{ intersects pixel } (i,j)) $$
In practice, this is implemented via interpolation (e.g., bilinear) to approximate the integral along the ray [S2, S4, unofficial].

**Parameters:**
- **theta:** Array of projection angles in degrees. Typical default: `np.linspace(0.0, 180.0, max(image.shape), endpoint=False)` [S6, unofficial]. In `skradon`, default is `torch.arange(180)` [S4, unofficial].
- **rs:** Detector positions. Default: linearly spaced from $-1$ to $1$ with length equal to image height [S3, unofficial].
- **circle:** Boolean. If True, assumes zero intensity outside the inscribed circle, adjusting sinogram width to `min(H, W)` [S4, unofficial].
- **preserve_range:** Boolean. If True, preserves original dtype range; otherwise converts to float $[0,1]$ [S4, unofficial].

**Steps:**
1. **Preprocessing:** Convert input image to float. If `preserve_range` is False, scale values according to `img_as_float` conventions (e.g., uint8 $[0,255]$ to $[0,1]$) [S4, unofficial].
2. **Angle Setup:** If `theta` is None, generate angles from $0$ to $179$ degrees [S4, unofficial].
3. **Geometry Setup:** If `circle` is True, calculate the radius $r = \min(H, W) // 2$ and mask pixels outside the inscribed circle to zero [S4, unofficial].
4. **Projection Loop (for each angle $\theta_k$ in `theta`):**
   a. **Affine Transformation:** Construct a $2 \times 3$ affine matrix representing rotation by $\theta_k$ and translation to center the projection axis [S4, unofficial].
   b. **Grid Generation:** Create a sampling grid using `affine_grid` based on the affine matrix and image dimensions. `align_corners=False` is standard [S4, unofficial].
   c. **Sampling:** Sample the image values at the grid coordinates using `grid_sample` (bilinear interpolation) [S4, unofficial].
   d. **Accumulation:** Sum along the vertical axis to obtain the projection profile and store it in the sinogram tensor [S4, unofficial].
5. **Post-processing:** If `circle` is True, pad the sinogram dimensions to square to handle inscribed circle geometry [S4, unofficial].
6. **Output:** Return the sinogram tensor [S4, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import torch
import torch.nn.functional as F

def radon_transform(image: torch.Tensor, theta: torch.Tensor = None, circle: bool = False, preserve_range: bool = False) -> torch.Tensor:
    """
    Computes the forward Radon transform (sinogram) of a batch of images.
    
    Args:
        image: Tensor of shape (N, C, H, W).
        theta: Tensor of angles in degrees. Default: arange(180).
        circle: If True, assumes zero outside inscribed circle.
        preserve_range: If False, converts image to float [0, 1].
        
    Returns:
        sinogram: Tensor of shape (N, C, D, A).
    """
    # 1. Preprocessing
    if not preserve_range:
        image = image.float()
        if image.dtype == torch.uint8:
            image = image / 255.0
        elif image.max() > 1.0:
            image = image / image.max()
    else:
        image = image.float()

    N, C, H, W = image.shape
    device = image.device

    # 2. Angle Setup
    if theta is None:
        theta = torch.arange(180, dtype=torch.float32, device=device)
    else:
        theta = theta.to(device=device, dtype=torch.float32)
    A = theta.shape[0]

    # 3. Detector Positions
    if circle:
        D = min(H, W)
    else:
        D = H

    # 4. Initialize Sinogram
    sinogram = torch.zeros((N, C, D, A), dtype=image.dtype, device=device)

    # 5. Loop over angles
    for k, angle_deg in enumerate(theta):
        angle_rad = torch.deg2rad(angle_deg)
        cos_a = torch.cos(angle_rad)
        sin_a = torch.sin(angle_rad)

        # Construct 2x3 affine matrix for rotation by -theta around center (0.5, 0.5)
        tx = 0.5 - 0.5 * cos_a - 0.5 * sin_a
        ty = 0.5 + 0.5 * sin_a - 0.5 * cos_a

        theta_mat = torch.zeros((N, 2, 3), device=device)
        theta_mat[:, 0, 0] = cos_a
        theta_mat[:, 0, 1] = sin_a
        theta_mat[:, 0, 2] = tx
        theta_mat[:, 1, 0] = -sin_a
        theta_mat[:, 1, 1] = cos_a
        theta_mat[:, 1, 2] = ty

        # Generate sampling grid
        grid = F.affine_grid(theta_mat, size=(N, C, H, W), align_corners=False)

        # Sample image values
        rotated_image = F.grid_sample(image, grid, mode='bilinear', padding_mode='zeros', align_corners=False)

        # Sum along vertical axis (height) to get projection profile
        profile = torch.sum(rotated_image, dim=2) # Shape: (N, C, W)

        # Map profile to detector positions
        if W == D:
            sinogram[:, :, :, k] = profile
        else:
            # Interpolate profile to D detectors [G]
            profile_resized = F.interpolate(profile.unsqueeze(2), size=(D), mode='linear', align_corners=False).squeeze(2)
            sinogram[:, :, :, k] = profile_resized

    # 6. Post-processing: Circle padding
    if circle:
        # Pad sinogram to square dimensions if needed [G]
        pass

    return sinogram
```

## Subtopics

### [Filtered Back-Projection (FBP)](11308-filtered-back-projection-fbp.md)
Filtered Back-Projection (FBP) is a direct tomographic reconstruction algorithm that recovers a 2D image from its measured projection profiles (sinogram). It solves the inverse Radon transform by applying a high-pass frequency-domain filter to the projections to correct for the inherent low-pass blurring of simple back-projection, followed by integrating the filtered data along sine-shaped paths. Unlike Fourier-based reconstruction methods which require interpolation between Cartesian and polar grids (leading to $O(N^2)$ complexity or interpolation artifacts), FBP operates directly on the projections with a time complexity of $O(N \log N)$ due to the use of Fast Fourier Transforms (FFT). It is the most popular algorithm for computed tomography.

### Single quadrant Radon transform
A variant that utilizes projections from only one quadrant ($0$ to $90$ degrees) to reduce computational overhead, exploiting a four-fold redundancy [S7, unofficial]. It computes back-projections by treating them as real and imaginary parts of a complex number and averaging individual back-projections [S7, unofficial]. It is chosen when computational resources are limited, but requires the quadrature approach to avoid poor reconstruction quality [S7, unofficial]. The quadrature Radon transform yields better results numerically and visually compared to the conventional Radon transform [S7, unofficial].

### Differentiable Radon Transform for Analysis-by-Synthesis
A differentiable 3D Radon transform used to project LAC volumes into sinogram space, synthesizing CT measurements [S8]. It allows backpropagation of gradients through the projection operation to update the INR and motion field parameters [S8]. Optimization updates INR weights ($\phi$) and motion field coefficients ($C$) via gradient descent (ADAM optimizer, learning rate $0.001$) to minimize the loss function by backpropagating derivatives through the differentiable Radon transform [S8]. The loss function is $\min_{\phi, C} \lambda_1 \| R_{\theta(t)}(\sigma(x, y, z, t)) - R_{\theta(t)}(GT) \|_1 + \lambda_2 TV(C)$, where $t \in [0, 1]$ [S8]. The TV Regularization Weight $\lambda_2$ is set to $0.001$; higher values create smoother warp fields but may underfit complex motion, while low values allow complex motion but are prone to noise [S8].

## Operating characteristics
- **Time Complexity:** $O(A \cdot H \cdot W)$ for a single image, where $A$ is the number of angles, $H$ height, $W$ width. The rotation and summation steps dominate [S1, S4, unofficial].
- **Memory Complexity:** $O(D \cdot A)$ for the sinogram storage, where $D$ is the number of detectors [S4, unofficial].
- **Performance:** Efficient for moderate resolutions. GPU acceleration via PyTorch allows for batch processing [S4, unofficial].
- **Strengths:** Differentiable when implemented with `grid_sample`, linear, and standard for forward projection in CT [S4, S6, unofficial].
- **Limitations:** Sensitive to aliasing if angles are sparse; requires interpolation for non-integer pixel shifts [S3, S4, unofficial].
- **When to use:** When generating synthetic sinograms for training or when the forward model is needed for differentiable tomography [S4, unofficial].
- **When not to use:** When only reconstruction is needed (use FBP or iterative methods) [S2, unofficial].

## Background
The Radon transform was introduced by Austrian mathematician Johann Radon in 1917 [S1, S2, unofficial], though some sources cite 1919 [S5, unofficial]. It provides the foundational link between projections and pixel values in tomographic reconstruction [S2, S6, unofficial]. The discrete implementation is widely used in medical imaging and industrial CT [S1, S4, unofficial]. The `skimage` implementation is based on Justin K. Romberg's code [S4, unofficial]. The differentiable `skradon` implementation extends this to PyTorch for differentiable inverse rendering [S4, unofficial].

## Sources
[S1] Reconstruction Tomographique Mojette, https://theses.hal.science/tel-00426920v1, fetched 2026-10-26 (unofficial)
[S2] Élaboration d'une méthode tomographique de reconstruction 3D en vélocimétrie par image de particules basée sur les processus ponctuels marqués, https://hal.science/tel-01292656v3, fetched 2026-10-26 (unofficial)
[S3] guibo-tech/Tomographic-Reconstruction, https://github.com/guibo-tech/Tomographic-Reconstruction, fetched 2026-10-26 (unofficial)
[S4] tomluetjen/torchskradon, https://github.com/tomluetjen/torchskradon, fetched 2026-10-27 (unofficial)
[S5] Précision et qualité en reconstruction tomographique : algorithmes et applications, https://doi.org/10.70675/d2bc4875z6a5az46afz9289z2fe16b3a43f3, fetched 2026-10-30 (unofficial)
[S6] plot_radon_transform.rst.txt, https://scikit-image.org/docs/stable/_sources/auto_examples/transform/plot_radon_transform.rst.txt, fetched 2026-10-30 (unofficial)
[S7] Quadrature Radon Transform for Smoother  Tomographic Reconstruction, https://core.ac.uk/download/234644432.pdf, fetched 2026-10-27 (unofficial)
[S8] Dynamic CT Reconstruction from Limited Views with Implicit Neural Representations and Parametric Motion Fields, https://doi.org/10.1109/iccv48922.2021.00226, fetched 2026-10-26
