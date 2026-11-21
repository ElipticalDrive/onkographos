# Synthetic CT Data Generation

Also called: Elliptical Phantom, Phantom Generation

## Overview
Synthetic CT Data Generation creates labeled training pairs for computed tomography reconstruction algorithms by procedurally generating ground-truth phantoms, simulating their projection data (sinograms), and reconstructing them. It solves the data scarcity problem in medical imaging by providing infinite, perfectly aligned input-target pairs without requiring real patient scans. It differs from [Voxel Grid Generation & Expansion](2979-voxel-grid-generation-expansion.md) by focusing on the full forward-backward pipeline (projection + noise + reconstruction) rather than just voxel occupancy, and from [Scene Data Loading & Resizing](3420-scene-data-loading-resizing.md) by generating data from mathematical primitives rather than loading existing assets.

## How it works
**Inputs:**
- $n$: Image dimension (square grid size). Typical values: $64$ or $128$ [S1, unofficial].
- $N_{ell}$: Number of ellipses to generate. Range: $5$ to $20$ [S1, unofficial].
- $\theta_{noise}$: Noise parameter for Poisson-like noise addition. Range: $100$ to $10000$ [S1, unofficial].
- $opacity$: Fill intensity for ellipses. Default: $0.1$ [S1, unofficial].
- $grad$: Boolean flag determining the target output. If true, target is the reconstructed image; if false, target is the noisy sinogram [S1, unofficial].

**Outputs:**
- A tuple $(f_{input}, f_{target})$ or $(f_{input}, g_{noisy})$ depending on the `grad` flag [S1, unofficial].

**Data Structures:**
- $f_{true}$: A 2D numpy array of shape $(n, n)$ representing the ground truth phantom.
- $g$: A 2D numpy array representing the clean sinogram.
- $g_{noisy}$: A 2D numpy array representing the noisy sinogram.
- $f_{BP}$: A 2D numpy array representing the reconstructed image using Filtered Back Projection (FBP).

**Steps:**
1. **Initialize Phantom:** Create a zero-filled 2D array $f_{true}$ of size $(n, n)$ [S1, unofficial].
2. **Generate Ellipses:** Randomly select $N_{ell}$ ellipses. For each ellipse, randomly sample:
   - Center coordinates $(x, y)$ within the image bounds.
   - Semi-major and semi-minor axes $(a, b)$.
   - Orientation angle $\theta_{ell}$ [S1, unofficial].
3. **Fill Phantom:** For each ellipse, compute the pixel indices belonging to the ellipse shape and set their values in $f_{true}$ to $opacity$. The boundary is determined using standard rasterization (e.g., `skimage.draw.ellipse`) [S1, unofficial].
4. **Normalize:** Normalize $f_{true}$ by dividing all pixel values by the maximum value in the array to ensure a consistent dynamic range [S1, unofficial].
5. **Create Sinogram:** Compute the forward projection of $f_{true}$ to generate the clean sinogram $g$. This involves defining projection geometry (angles, samples) and using a projector (e.g., ASTRA toolbox) [S1, unofficial].
6. **Add Noise:** Apply Poisson-like noise to $g$ using a noise function parameterized by $\theta_{noise}$ to produce $g_{noisy}$. The function is typically `astra.functions.add_noise_to_sino(g, theta)` [S1, unofficial].
7. **Reconstruct:** Reconstruct the image from $g_{noisy}$ using Filtered Back Projection (FBP) to produce $f_{BP}$. The reconstruction uses the inverse of the projection geometry [S1, unofficial].
8. **Return Pair:**
   - If `grad` is true, return $(f_{BP}, f_{true})$. This trains the network to map a noisy reconstruction to the ground truth [S1, unofficial].
   - If `grad` is false, return $(f_{BP}, g_{noisy})$ or potentially $(g_{noisy}, f_{true})$ depending on specific implementation intent, but the source specifies returning the reconstructed image and the noisy sinogram or true image [S1, unofficial]. *Note: The source code snippet shows reconstruction from noisy sinogram, implying the input to a downstream task is likely the reconstruction or the sinogram itself.*

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import numpy as np
from skimage.draw import ellipse
import astra

def generate_synthetic_ct_data(n, num_ellipses=10, theta_noise=1000, opacity=0.1, grad=True):
    """
    Generates synthetic CT training data.
    
    Args:
        n (int): Image dimension (square).
        num_ellipses (int): Number of ellipses (5-20).
        theta_noise (int): Noise parameter (100-10000).
        opacity (float): Ellipse fill intensity.
        grad (bool): If True, target is ground truth; else target is noisy sinogram/recon.
        
    Returns:
        tuple: (input_data, target_data)
    """
    # 1. Initialize zero image
    f_true = np.zeros((n, n), dtype=np.float32)
    
    # 2 & 3. Randomly select and fill ellipses
    for _ in range(num_ellipses):
        # Random parameters
        center_x = np.random.randint(0, n)
        center_y = np.random.randint(0, n)
        a = np.random.randint(1, n // 4)  # Semi-major axis
        b = np.random.randint(1, n // 4)  # Semi-minor axis
        theta = np.random.uniform(0, np.pi) # Orientation
        
        # Get ellipse coordinates
        rr, cc = ellipse(center_y, center_x, a, b, orientation=theta, shape=(n, n))
        
        # Fill with opacity
        f_true[rr, cc] = opacity
        
    # 4. Normalize by max value
    max_val = np.max(f_true)
    if max_val > 0:
        f_true = f_true / max_val
        
    # 5. Create sinogram
    # Define geometry (simplified; actual implementation requires specific ASTRA geometry setup)
    # proj_geom, vol_geom, projector_id are assumed to be created based on n and projection angles
    proj_geom = astra.create_proj_geom('parallel', 1.0, n, np.linspace(0, np.pi, n, endpoint=False))
    vol_geom = astra.create_vol_geom(n, n)
    projector_id = astra.create_projector('linear', proj_geom, vol_geom)
    
    # Create sinogram ID and data
    g_id = astra.data2d.create('-sino', proj_geom, f_true)
    # Forward project to get clean sinogram g
    # Note: astra.create_projector creates the operator; we need to project f_true
    # Using the projector to compute sinogram
    g = astra.create_sino(f_true, proj_geom) # Simplified; actual ASTRA usage might differ
    
    # 6. Add noise
    g_noisy = astra.functions.add_noise_to_sino(g, theta_noise)
    
    # 7. Reconstruct via FBP
    g_noisy_id = astra.data2d.create('-sino', proj_geom, g_noisy)
    f_BP = BP_recon(vol_geom, projector_id, g_noisy_id, 'FBP')
    
    # 8. Return based on grad flag
    if grad:
        return f_BP, f_true
    else:
        return f_BP, g_noisy

def BP_recon(vol_geom, projector_id, sino_id, method):
    """
    Wrapper for ASTRA FBP reconstruction.
    """
    if method == 'FBP':
        # ASTRA toolbox FBP reconstruction
        # Note: ASTRA's FBP is often accessed via specific reconstruction algorithms
        # This is a placeholder for the specific ASTRA call structure
        recon_id = astra.recon.FBP(sino_id, vol_geom)
        return astra.data2d.get(recon_id)
    else:
        raise ValueError("Only FBP is supported in this basic generator")
```

## Subtopics

### Synthetic Phantom Generation
Generates synthetic CT phantoms consisting of spheres rather than ellipses for testing reconstruction algorithms. It uses a dedicated `generate_sphere_phantom` function to create a 3D volume of spheres. This variant is chosen when spherical symmetry is required for algorithm validation or when testing 3D reconstruction methods like FDK [S2, unofficial].

## Operating characteristics
- **Time Complexity:** Linear with respect to the number of pixels $n^2$ for phantom generation and the number of projections for sinogram creation. Reconstruction is typically $O(n^2 \cdot N_{angles})$ for FBP.
- **Memory Complexity:** $O(n^2)$ for the phantom and sinogram storage.
- **Performance:** Highly efficient for generating large datasets. The use of optimized libraries like ASTRA ensures fast projection and reconstruction.
- **Strengths:** Provides perfect ground truth; infinite scalability; customizable noise levels and phantom complexity.
- **Limitations:** Elliptical phantoms are simplistic and do not represent complex anatomical structures; noise model is simplified (Poisson-like) and may not capture all real-world scanner artifacts.
- **When to use:** Ideal for initial algorithm development, debugging, and training deep learning models where labeled data is scarce.
- **When not to use:** For final validation of clinical-grade reconstruction algorithms, where realistic anatomical phantoms and complex noise models are required.

## Background
The method originates from research in inverse problems for CT scans, specifically utilizing the ASTRA toolbox for GPU-accelerated projection and reconstruction. It was popularized in repositories focusing on deep learning for CT reconstruction, such as `imanfs/inverse-problems-CT-scans` and `ayush-chauhan-1202/physics-aware-ct-reconstruction` [S1, S2, unofficial]. The approach leverages the mathematical simplicity of ellipses and spheres to create analytically known ground truths, facilitating the evaluation of reconstruction errors (e.g., MSE) [S1, S2, unofficial].

## Sources
[S1] imanfs/inverse-problems-CT-scans, https://github.com/imanfs/inverse-problems-CT-scans, fetched 2026-10-27 (unofficial)
[S2] ayush-chauhan-1202/physics-aware-ct-reconstruction, https://github.com/ayush-chauhan-1202/physics-aware-ct-reconstruction, fetched 2026-10-27 (unofficial)
