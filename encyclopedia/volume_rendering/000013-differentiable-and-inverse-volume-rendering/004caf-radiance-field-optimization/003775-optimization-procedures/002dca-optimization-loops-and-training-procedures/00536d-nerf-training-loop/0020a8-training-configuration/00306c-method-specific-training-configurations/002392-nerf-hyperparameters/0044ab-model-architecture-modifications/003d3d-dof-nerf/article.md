# DoF-NeRF

Also called: Depth-of-Field Meets Neural Radiance Fields

## Overview
DoF-NeRF is a neural rendering framework that extends the standard Neural Radiance Field (NeRF) architecture to simulate the depth-of-field (DoF) effects of a physical lens, allowing it to operate on and synthesize views with arbitrary focus configurations [S1]. Unlike standard NeRF or methods like RawNeRF and NeRFocus which assume all-in-focus inputs, DoF-NeRF explicitly models the aperture and focus distance, enabling the optimization of a clear, all-in-focus 3D scene representation from shallow DoF inputs [S1]. It achieves this by combining optical models with volume rendering to compute observed colors based on the Circle of Confusion (CoC) diameter [S1].

## How it works
DoF-NeRF modifies the standard NeRF rendering pipeline to account for light scattering caused by finite aperture size. The method introduces two learnable parameters per training view: the aperture parameter $K$ and the focus distance $F$ [S1]. These are jointly optimized with the NeRF network parameters [S1].

### Inputs and Outputs
*   **Inputs:** A set of $N$ training images $I = \{I_1, \dots, I_N\}$ with known camera poses (generated via COLMAP) [S1].
*   **Outputs:** A trained NeRF model $G_\Theta$ capable of rendering all-in-focus novel views, along with optimized aperture parameters $\hat{K}$ and focus distances $\hat{F}$ [S1].

### Data Structures
The method extends the standard NeRF MLPs. The standard NeRF architecture consists of a coarse network and a fine network [Parent]. DoF-NeRF retains this structure but adds learnable scalar parameters for each view:
*   **Aperture Parameters:** $\hat{K} = \{\hat{K}_1, \dots, \hat{K}_N\}$, where $\hat{K}_n \in \mathbb{R}$. Initialized as learnable parameters.
*   **Focus Distances:** $\hat{F} = \{\hat{F}_1, \dots, \hat{F}_N\}$, where $\hat{F}_n \in \mathbb{R}$. Initialized as learnable parameters.
*   **NeRF Model:** $G_\Theta$, containing the coarse and fine MLPs as defined in the parent article [Parent].

### Formulas
The core physical model relies on the Circle of Confusion (CoC). For a spatial point at object distance $h_t$, the simplified CoC diameter $\delta(h_t)$ is defined as [S1]:

$$
\delta(h_t) = K \times \left| \frac{1}{F} - \frac{1}{h_t} \right|
$$

where $K = f \times D$ is the aperture parameter (product of focal length $f$ and aperture diameter $D$) [S1].

The DoF effect is implemented via a "concentrate-and-scatter" process involving two steps:
1.  **Concentration:** Radiance from spatial points is accumulated along the center ray.
2.  **Scattering:** The center ray accepts scattered radiance from neighboring rays based on the CoC size [S1].

The scattered radiance $\hat{I}_{scatter}(p)$ at a pixel $p$ sums contributions from $M$ neighboring pixels $p_j$ [S1]:

$$
\hat{I}_{scatter}(p) = \sum_{j=1}^{M} \sum_{i=1}^{N_s} \mathbb{1}_{\delta(h'_i) > 2 \|p - p_j\|^2} T'_{ij} \hat{C}(p_j, h'_i)
$$

where $N_s$ is the number of samples, $T'_{ij}$ is the transmittance, and $\hat{C}$ is the color [S1].

The scattered radiance at the concentration point $\hat{C}^s(p)$ is calculated as [S1]:

$$
\hat{C}^s(p) = \frac{\sum_{i=1}^{N_s} K_{volume}(r_p, h_i) c(r_p(h_i), d)}{\pi \delta_c^2(p)/4}
$$

where $K_{volume}$ represents the volume rendering kernel, $c$ is the color, and $\delta_c(p)$ is the CoC radius at pixel $p$ [S1].

The final observed color $\hat{I}(p)$ for $P$ rays is computed as [S1]:

$$
\hat{I}(p) = \sum_{j=1}^{P} \mathbb{1}_{\delta_c(p) > 2 \|p - p_j\|^2} \hat{C}^s(p_j)
$$

The optimization loss $L$ is the standard photometric error between synthesized views $\hat{I}$ and observations $I$, summed over all $N$ input images [S1]:

$$
L = \sum_{n=1}^{N} \| I_n - \hat{I}_n \|_2^2
$$

### Parameters
*   **Aperture Parameter ($K$):** Learnable scalar per view. Not given in the sources.
*   **Focus Distance ($F$):** Learnable scalar per view. Not given in the sources.
*   **Patch Size ($N_{patch}$):** Size of the patch for ray selection. Not given in the sources.
*   **Anchor Spacing ($N_{anchor}$):** Spacing between anchors on the imaging plane. Not given in the sources.
*   **Image Resolution:** 497×331 for real-world datasets; 504×378 for synthetic datasets [S1].

### Pseudocode
The training follows a two-stage optimization process [S1]. Stage 1 trains the NeRF with aperture $K=0$ (pinhole camera) using random rays. Stage 2 jointly optimizes NeRF, $K$, and $F$ using concentrate-and-scatter rendering with patch-based ray selection [S1].

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import torch
import torch.nn as nn

# Assume NeRF_Model is defined as per [Parent]
# Assume Volume_Rendering is defined as per [Parent]

class DoFNeRF:
    def __init__(self, N_images, nerf_model):
        self.N = N_images
        self.G = nerf_model  # NeRF Model G_Theta
        
        # Learnable parameters for each view
        # Shape: (N, 1)
        self.K = nn.Parameter(torch.randn(N_images, 1), requires_grad=True)
        self.F = nn.Parameter(torch.randn(N_images, 1), requires_grad=True)
        
        # Hyperparameters
        self.N_pretrain = 10000  # Not given in sources, placeholder for stage 1 iterations
        self.N_iter = 100000     # Not given in sources, placeholder for total iterations
        self.N_anchor = 10       # Not given in sources
        self.N_patch = 16        # Not given in sources

    def concentration_step(self, rays, K_val, F_val):
        """
        Computes the concentration of radiance along rays.
        Returns scattered radiance C^s and CoC radius delta_c.
        """
        # 1. Sample points along rays and query NeRF for density and color
        # This is a standard volume rendering step, adapted to return intermediate values
        # for the concentrate step.
        # Let's assume we get samples h_i, transmittance T_i, and color c_i
        # from the NeRF model for the given rays.
        
        # Standard volume rendering to get the "concentrated" color
        # Note: The source defines C^s(p) using a volume kernel K_volume.
        # In implementation, this is typically the accumulated color from the ray.
        
        # Calculate CoC radius for each sample point
        # h_t is the depth (distance from camera) of the sample point
        # delta(h_t) = K * |1/F - 1/h_t|
        # We need to compute this for all samples along all rays.
        
        # For simplicity, we assume the NeRF model or a helper provides
        # the necessary depth and transmittance values.
        # In a real implementation, this requires modifying the volume renderer
        # to output per-sample depths and transmittances.
        
        # Placeholder for the actual concentration calculation logic
        # which involves summing contributions weighted by the volume kernel.
        # The source formula: C^s(p) = sum(K_volume * c) / (pi * delta_c^2 / 4)
        
        # Since exact implementation of K_volume is not detailed beyond the formula,
        # we assume a standard volume rendering accumulation is performed,
        # then divided by the CoC area factor.
        
        # Let's assume we have a function that returns the accumulated color
        # and the CoC radius at the pixel level.
        
        # This step is complex and relies on specific volume rendering internals.
        # We will outline the logic based on the formulas.
        
        # 1. Get samples along rays
        # 2. Compute delta_c for each sample based on depth h_t, K, F
        # 3. Accumulate color contributions, weighted by transmittance and density
        # 4. Normalize by CoC area
        
        # Note: The source indicates this is done per patch.
        pass

    def scatter_step(self, C_s, delta_c, rays, patch_indices):
        """
        Computes the scattered radiance by aggregating from neighboring pixels.
        """
        # For each pixel p in the patch:
        # I(p) = sum_{j in neighbors} 1_{delta_c(p) > 2 ||p - p_j||^2} * C^s(p_j)
        
        # This requires accessing C^s from neighboring pixels.
        # The condition checks if the CoC diameter is large enough to cover the distance.
        pass

    def patch_rays(self, images):
        """
        Selects rays based on patch-based selection.
        """
        # Construct anchors every N_anchor pixels
        # Randomly choose one patch
        # Return rays passing through pixels in that patch
        pass

    def train(self, images):
        optimizer = torch.optim.Adam(list(self.G.parameters()) + [self.K, self.F], lr=1e-4)
        
        for i in range(self.N_iter):
            if i < self.N_pretrain:
                # Stage 1: Pinhole camera (K=0)
                # Set K to 0 for this stage
                self.K.data.fill_(0.0)
                
                rays, target_images = self.random_rays(images)
                # Standard volume rendering
                rendered_images = self.volume_rendering(self.G, rays)
                
                loss = torch.mean((rendered_images - target_images) ** 2)
                loss.backward()
                optimizer.step()
                optimizer.zero_grad()
            else:
                # Stage 2: Concentrate-and-Scatter
                rays, target_images = self.patch_rays(images)
                
                # Concentration step
                C_s, delta_c = self.concentration_step(rays, self.K, self.F)
                
                # Scatter step
                rendered_images = self.scatter_step(C_s, delta_c, rays, None)
                
                loss = torch.mean((rendered_images - target_images) ** 2)
                loss.backward()
                optimizer.step()
                optimizer.zero_grad()

    def random_rays(self, images):
        # Generate random rays for the given images
        pass

    def volume_rendering(self, model, rays):
        # Standard NeRF volume rendering
        pass
```

### Edge Cases and Pitfalls
*   **Divergence:** Using random rays for DoF simulation in the initial stages can lead to divergence or sub-optimal results due to ray gathering [S1]. This is mitigated by the two-stage optimization process [S1].
*   **Computational Cost:** Computing rays for the whole imaging plane in each iteration is computationally unacceptable [S1]. Patch-based ray selection is used as a compromise [S1].
*   **Blurriness:** Vanilla NeRF on shallow DoF inputs tends to generate blurry renderings and miss texture details [S1]. DoF-NeRF addresses this by explicitly modeling aperture and focus [S1].

## Operating characteristics
*   **Time Complexity:** Not given in the sources. However, the method uses patch-based ray selection to reduce the cost compared to computing rays for the whole imaging plane [S1].
*   **Memory Complexity:** Not given in the sources.
*   **Measured Performance:**
    *   **Real-world dataset (amiya scene):** PSNR 28.311 (vs NeRF 26.924), SSIM 0.9289 (vs 0.9092), LPIPS 0.1370 (vs 0.1633) [S1].
    *   **Synthetic dataset (room scene):** PSNR 29.443 (vs NeRF 26.668), SSIM 0.9135 (vs 0.8743), LPIPS 0.1502 (vs 0.1961) [S1].
*   **Strengths:**
    *   Can handle shallow DoF inputs and produce clear all-in-focus novel views [S1].
    *   Allows direct manipulation of DoF effects by adjusting virtual aperture and focus parameters [S1].
    *   Plug-and-play with NeRF-based frameworks [S1].
*   **Limitations:**
    *   Requires optimization of additional parameters ($K$ and $F$) per view [S1].
    *   Assumes ideal optical system, neglecting ray distortions caused by the lens [S1].
*   **When to use:** When input images have shallow depth-of-field and the goal is to reconstruct an all-in-focus 3D scene or simulate DoF effects [S1].
*   **When not to use:** When inputs are already all-in-focus and DoF simulation is not required, standard NeRF may be sufficient and simpler [S1].

## Background
DoF-NeRF was introduced by Zijin Wu, Juewen Peng, and Weicai Zhong from Huazhong University of Science and Technology and Huawei [S1]. The paper was published in the Proceedings of the ACM on Multimedia 2022 (MM '22) in Lisboa, Portugal [S1]. The work addresses the limitation of standard NeRF, which assumes a linear pinhole camera model and all-in-focus inputs, making it unsuitable for real-world images with finite depth-of-field [S1]. It builds upon the principles of geometric optics to model the aperture and CoC [S1].

## Sources
[S1] DoF-NeRF: Depth-of-Field Meets Neural Radiance Fields, https://doi.org/10.1145/3503161.3548088, fetched 2026-10-29
