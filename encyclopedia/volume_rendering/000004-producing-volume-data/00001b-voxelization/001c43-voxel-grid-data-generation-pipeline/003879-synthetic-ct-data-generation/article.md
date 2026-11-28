# Synthetic CT Data Generation

Also called: Elliptical Phantom, Phantom Generation

## Overview
Synthetic CT Data Generation creates labeled training pairs for computed tomography reconstruction algorithms by procedurally generating ground-truth phantoms, simulating their projection data (sinograms), and reconstructing them [S1, unofficial]. It differs from [Voxel Grid Generation & Expansion](2979-voxel-grid-generation-expansion.md) by focusing on the full forward-backward pipeline (projection + noise + reconstruction) rather than just voxel occupancy [S1, unofficial].

## How it works
**Inputs:**
- $n$: Image dimension (square grid size). Typical values: $64$ or $128$ [S1, unofficial].
- $\theta$: Noise parameter for Poisson-like noise addition. Typical range: $100$ to $10000$ [S1, unofficial].
- $opacity$: Fill intensity for ellipses. Default: $0.1$ [S1, unofficial].
- $grad$: Boolean flag determining the target output [S1, unofficial].

**Outputs:**
- A tuple $(f_{BP}, f_{true})$ or $(f_{BP}, g_{Noisy})$ depending on the `grad` flag [S1, unofficial].

**Data Structures:**
- $f_{true}$: A 2D array of shape $(n, n)$ representing the ground truth phantom [S1, unofficial].
- $g$: A 2D array representing the clean sinogram [S1, unofficial].
- $g_{Noisy}$: A 2D array representing the noisy sinogram [S1, unofficial].
- $f_{BP}$: A 2D array representing the reconstructed image [S1, unofficial].

**Steps:**
1. Initialize a zero-filled 2D array $f_{true}$ of size $(n, n)$ [S1, unofficial].
2. Randomly select 5 to 20 ellipses [S1, unofficial]. For each, pick center $(x, y)$, axes $(a, b)$, and orientation $\theta_{orient}$ [S1, unofficial].
3. Fill pixels using `skimage.draw.ellipse` and set their values in $f_{true}$ to $opacity$ [S1, unofficial].
4. Normalize $f_{true}$ by dividing all pixel values by the maximum value in the array [S1, unofficial].
5. Create the clean sinogram $g$ via `create_sinogram` [S1, unofficial].
6. Add noise via `astra.functions.add_noise_to_sino(g, theta)` to produce $g_{Noisy}$ [S1, unofficial].
7. Reconstruct via `BP_recon(..., 'FBP')` to produce $f_{BP}$ [S1, unofficial].
8. Return $(f_{BP}, f_{true})$ or $(f_{BP}, g_{Noisy})$ depending on the `grad` flag [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import numpy as np
from skimage.draw import ellipse
import astra

def generate_synthetic_ct_data(n, theta=1000, opacity=0.1, grad=True):
    # 1. Initialize zero image
    f_true = np.zeros((n, n), dtype=np.float32)

    # 2 & 3. Randomly select 5 to 20 ellipses and fill
    num_ellipses = np.random.randint(5, 21)
    for _ in range(num_ellipses):
        center_x = np.random.randint(0, n)
        center_y = np.random.randint(0, n)
        a = np.random.randint(1, n // 4)
        b = np.random.randint(1, n // 4)
        theta_orient = np.random.uniform(0, np.pi)
        rr, cc = ellipse(center_y, center_x, a, b, orientation=theta_orient, shape=(n, n))
        f_true[rr, cc] = opacity

    # 4. Normalize by max value
    f_true = f_true / np.max(f_true)

    # 5. Create sinogram
    proj_geom = astra.create_proj_geom('parallel', 1.0, n, np.linspace(0, np.pi, n, endpoint=False))
    vol_geom = astra.create_vol_geom(n, n)
    projector_id = astra.create_projector('linear', proj_geom, vol_geom)
    g = astra.create_sino(f_true, proj_geom)

    # 6. Add noise
    g_Noisy = astra.functions.add_noise_to_sino(g, theta)

    # 7. Reconstruct via FBP
    g_Noisy_id = astra.data2d.create('-sino', proj_geom, g_Noisy)
    f_BP = BP_recon(vol_geom, projector_id, g_Noisy_id, 'FBP')

    # 8. Return pair
    if grad:
        return f_BP, f_true
    else:
        return f_BP, g_Noisy

def BP_recon(vol_geom, projector_id, sino_id, method):
    # Wrapper for ASTRA FBP reconstruction [G]
    if method == 'FBP':
        recon_id = astra.recon.FBP(sino_id, vol_geom)
        return astra.data2d.get(recon_id)
    raise ValueError("Unsupported method")
```

## Subtopics

### Synthetic Phantom Generation
Generates synthetic CT phantoms consisting of spheres for testing reconstruction algorithms [S2, unofficial]. It uses a dedicated `generate_sphere_phantom` function to create a 3D volume of spheres [S2, unofficial].

## Operating characteristics

## Background
The method originates from the `imanfs/inverse-problems-CT-scans` repository, which implements synthetic CT data generation using the ASTRA toolbox for GPU-accelerated projection and reconstruction [S1, unofficial].

## Sources
[S1] imanfs/inverse-problems-CT-scans - GitHub, https://github.com/imanfs/inverse-problems-CT-scans, fetched 2026-10-27 (unofficial)
[S2] ayush-chauhan-1202/physics-aware-ct-reconstruction, https://github.com/ayush-chauhan-1202/physics-aware-ct-reconstruction, fetched 2026-10-27 (unofficial)
