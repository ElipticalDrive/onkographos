# AReT (Anatomy-Regularized TensoRF)

Also called: AReT NeRF, AReT framework, AReT model, Anatomy-Regularized TensoRF, Prior TensoRF v2, proposed AReT model

## Overview
AReT is a scene-specific (patient-specific) inverse solver that reconstructs 3D lung nodule volumes from only three orthogonal Digitally Reconstructed Radiographs (DRRs: coronal, sagittal, axial) using a Tensorial Radiance Field (TensoRF) representation [S1, unofficial]. It solves the problem of sparse-view medical reconstruction where standard TensoRF fails due to a density-parameterization dead zone that suppresses gradients [S1, unofficial]. Unlike generative-prior approaches like MedNeRF, AReT is prior-free during training, relying instead on a corrected density initialization and anatomy-aware regularization ($\ell_1$ sparsity and Total Variation smoothness) to enforce physiologically plausible attenuation distributions in the air-dominated thoracic cavity [S1, unofficial].

## How it works
**Inputs and Outputs:**
*   **Inputs:** Three orthogonal DRRs (coronal, sagittal, axial) generated from a patient's ground-truth CT volume via Beer-Lambert forward projection [S1, unofficial].
*   **Outputs:** A reconstructed 3D density field (linear attenuation coefficient $\mu$) and a segmented nodule volume [S1, unofficial].

**Data Structures:**
*   **TensoRF Representation:** A VM-rank-16 TensoRF model decomposes the scene into low-rank tensor factors [S1, unofficial].
*   **Grid Resolution:** A $64^3$ grid is used for density queries, with voxel spacing approximately $[3.75, 1.875, 2.1875]$ mm [S1, unofficial].
*   **Memory:** A $64^3$ VM-rank-16 TensoRF requires approximately 14 GB VRAM (including gradient buffers) on an NVIDIA T4 [S1, unofficial].

**Parameters:**
*   **Density Shift ($\Delta$):** Set to $0$ (default in standard TensoRF is $-10$) [S1, unofficial].
*   **Scale ($s$):** Set to $1$ (default in standard TensoRF is $25$) [S1, unofficial].
*   **Regularization Weights:** $\lambda_1 = 1 \times 10^{-4}$ (sparsity), $\lambda_2 = 1 \times 10^{-5}$ (TV smoothness) [S1, unofficial].
*   **Optimizer:** Adam with learning rate $0.02$ for grid parameters and $0.002$ for MLP [S1, unofficial].
*   **Batch Size:** 2048 rays per iteration [S1, unofficial].
*   **Iterations:** 3000 total iterations [S1, unofficial].
*   **Alpha Mask Updates:** Deterministic updates at iterations 1500 and 2500 to prune near-empty voxels [S1, unofficial].
*   **Random Seeds:** `torch.manual_seed(0)` and `numpy.random.seed(0)` [S1, unofficial].

**Numbered Steps:**

1.  **CT Preprocessing and HU Calibration:**
    *   Load DICOM slices and sort by `ImagePositionPatient` Z-coordinate [S1, unofficial].
    *   Convert Hounsfield Units (HU) using `RescaleSlope` and `RescaleIntercept` [S1, unofficial].
    *   Apply lung-window clipping and linear normalization:
        $$ v_{norm} = \frac{HU - (-1000)}{400 - (-1000)} $$
        [S1, unofficial]

2.  **DRR Simulation and Camera Pose Assignment:**
    *   Generate three orthogonal DRRs (coronal, sagittal, axial) from the normalized CT volume using the Beer-Lambert forward model:
        $$ I(r) = I_0 \cdot \exp \left( - \int \mu(x) \, dx \right) $$
        where $\mu(x)$ is approximated by summing normalized HU values along the projection direction [S1, unofficial].
    *   Assign camera poses corresponding to the orthogonal directions [S1, unofficial].

3.  **AReT Training:**
    *   Initialize the TensoRF model with $\Delta = 0$ and $s = 1$ [S1, unofficial].
    *   For each iteration $i$ from 1 to 3000:
        *   Sample 2048 rays randomly [S1, unofficial].
        *   Render DRRs from the current TensoRF model [S1, unofficial].
        *   Compute Photometric Loss ($L_{photo}$): Mean Squared Error (MSE) between rendered DRRs and input DRRs [S1, unofficial].
        *   Compute Regularization Losses:
            *   Sparsity Loss ($L_{sparse}$): $\ell_1$ norm of the density field [S1, unofficial].
            *   Smoothness Loss ($L_{TV}$): Total Variation of the density field [S1, unofficial].
        *   Compute Total Loss:
            $$ L = L_{photo} + \lambda_1 L_{sparse} + \lambda_2 L_{TV} $$
            [S1, unofficial]
        *   Update TensoRF parameters using Adam optimizer [S1, unofficial].
        *   If $i \in \{1500, 2500\}$, update the alpha occupancy mask to prune near-empty voxels [S1, unofficial].

4.  **Volumetric Reconstruction and Nodule Extraction:**
    *   Query the converged density field on a $64^3$ grid using `getDenseAlpha` [S1, unofficial].
    *   Perform a percentile threshold sweep on the non-zero density distribution for $p \in \{85, 88, 90, 91, \dots, 99\}$ [S1, unofficial].
    *   For each threshold:
        *   Remove connected components with fewer than 10 voxels [S1, unofficial].
        *   Apply binary closing with a radius-2 spherical structuring element to fill boundary gaps [S1, unofficial].
        *   Identify connected components with equivalent sphere diameters in $[5, 80]$ mm [S1, unofficial].
    *   Select the candidate whose equivalent sphere diameter is closest to the LIDC-IDRI annotated diameter for that patient [S1, unofficial].
    *   Compute physical volume:
        $$ V = N_{voxels} \times \prod \Delta x_k $$
        where $\Delta x_k$ are the per-axis voxel spacings [S1, unofficial].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import torch
import numpy as np

def train_arect(ct_volume, annotations, patient_id):
    # 1. Preprocessing
    ct_sorted = sort_slices_by_z(ct_volume)
    ct_hu = apply_hu_calibration(ct_sorted)
    ct_normalized = (ct_hu - (-1000)) / (400 - (-1000))

    # 2. DRR Generation
    drrs = simulate_drrs(ct_normalized)
    camera_poses = get_orthogonal_poses()

    # 3. Model Initialization
    model = TensoRF(rank=16, grid_size=64)
    model.set_density_params(delta=0, s=1)
    optimizer = torch.optim.Adam(model.parameters(), lr=0.02)

    # 4. Training Loop
    for i in range(1, 3001):
        rays = sample_rays(2048, camera_poses)
        rendered_drrs = model.render(rays)

        l_photo = torch.mean((rendered_drrs - drrs) ** 2)
        density_field = model.get_density_field()
        l_sparse = torch.sum(torch.abs(density_field))
        l_tv = torch.sum(torch.abs(torch.diff(density_field, dim=0))) + \
               torch.sum(torch.abs(torch.diff(density_field, dim=1))) + \
               torch.sum(torch.abs(torch.diff(density_field, dim=2)))

        loss = l_photo + 1e-4 * l_sparse + 1e-5 * l_tv
        loss.backward()
        optimizer.step()
        optimizer.zero_grad()

        if i in [1500, 2500]:
            model.update_alpha_mask()

    return model.get_density_field()

def extract_nodule(density_field, annotated_diameter):
    candidates = []
    non_zero_mask = density_field > 0
    non_zero_values = density_field[non_zero_mask]

    for p in range(85, 100):
        threshold = np.percentile(non_zero_values, p)
        binary_mask = (density_field >= threshold).astype(np.uint8)

        # Noise removal: remove components < 10 voxels
        labeled, num = label_connected_components(binary_mask)
        valid_mask = np.zeros_like(binary_mask)
        for k in range(1, num + 1):
            if np.sum(labeled == k) >= 10:
                valid_mask[labeled == k] = 1

        # Boundary filling: binary closing with radius-2 spherical structuring element
        filled_mask = binary_closing(valid_mask, radius=2)

        # Size filtering: retain components with diameter in [5, 80] mm
        labeled_filled, num_filled = label_connected_components(filled_mask)
        for k in range(1, num_filled + 1):
            comp = (labeled_filled == k)
            diameter = compute_equivalent_sphere_diameter(comp)
            if 5 <= diameter <= 80:
                candidates.append((comp, diameter))

    best_candidate = min(candidates, key=lambda x: abs(x[1] - annotated_diameter))
    return compute_physical_volume(best_candidate[0])

# Standard helper implementations
def sort_slices_by_z(volume): return sorted(volume, key=lambda s: s.z) # [G]
def apply_hu_calibration(hu_data): return hu_data * slope + intercept # [G]
def simulate_drrs(ct_vol): return forward_project_beer_lambert(ct_vol) # [G]
def get_orthogonal_poses(): return [coronal_pose, sagittal_pose, axial_pose] # [G]
def sample_rays(n, poses): return random.sample(poses, n) # [G]
def label_connected_components(mask): return scipy.ndimage.label(mask) # [G]
def binary_closing(mask, radius): return scipy.ndimage.binary_closing(mask, structure=np.ones((2*radius+1,)*3)) # [G]
def compute_equivalent_sphere_diameter(mask):
    voxels = np.sum(mask)
    return (3 * voxels / (4 * np.pi)) ** (1/3) # [G]
def compute_physical_volume(mask):
    return np.sum(mask) * np.prod(voxel_spacings) # [G]
```

**Edge Cases and Pitfalls:**
*   **Density Parameterization Dead Zone:** Using the default TensoRF density shift of $\Delta = -10$ (and $s=25$) suppresses all learned density and prevents sparse-view reconstruction regardless of learning rate or regularization strategy [S1, unofficial]. Setting $\Delta = 0$ and $s=1$ restores gradient flow [S1, unofficial].
*   **Resolution Limits:** The $64^3$ grid yields voxel spacing $\approx 3.75$ mm, making sub-centimeter nodules ($< 10$ mm) unsuitable as they span fewer than 2.3 voxels per axis [S1, unofficial].
*   **Domain Gap:** The method uses simulated DRRs rather than real radiographs. Real radiographs exhibit scatter, detector noise, beam hardening, motion artifacts, and acquisition inconsistencies not present in simulation, creating a domain gap for clinical translation [S1, unofficial].
*   **Semi-Assisted Selection:** Final candidate selection relies on the annotated nodule diameter, which is clinically unavailable in a blind reconstruction scenario, introducing weak supervision during post-processing [S1, unofficial].

## Subtopics

### TensoRF
A tensor-based radiance field representation that decomposes the 3D scene into low-rank factors, offering memory efficiency compared to dense grids. AReT uses a VM-rank-16 configuration [S1, unofficial].

### MedNeRF
A generative-prior-guided NeRF approach that uses a population-level latent code from a GAN. AReT outperforms MedNeRF in sparse-view settings because MedNeRF's diffuse density field and scene-agnostic thresholding degrade patient-specific accuracy [S1, unofficial].

### Classical Sparse-View Methods (FDK, SART)
Require 20-50 views and produce streak artifacts with only three views. AReT works with three orthogonal projections [S1, unofficial].

## Operating characteristics
*   **Time Complexity:** Training takes approximately 8 minutes per patient on an NVIDIA T4 GPU [S1, unofficial].
*   **Memory Complexity:** A $64^3$ VM-rank-16 TensoRF requires ~14 GB VRAM. A $128^3$ grid exceeds the 16 GB VRAM limit of the T4 [S1, unofficial].
*   **Performance:** On LIDC-IDRI (19 patients, 14 nodules $\ge$ 10 mm), AReT achieves a Pearson correlation $r = 0.983$ ($p < 0.0001$), median absolute volumetric error 11.4%, and systematic bias $-77.3$ mm$^3$ [S1, unofficial].
*   **Strengths:** Works with only three orthogonal projections; prior-free patient-specific reconstruction; robust to sparse-view overfitting via anatomy-aware regularization.
*   **Limitations:** Requires simulated DRRs (domain gap with real radiographs); unsuitable for sub-centimeter nodules ($< 10$ mm) due to resolution limits; relies on annotated diameter for final candidate selection (semi-assisted) [S1, unofficial].
*   **When to use:** Sparse-view 3D lung nodule volumetry from orthogonal X-ray projections.
*   **When not to use:** Dense-view reconstruction; sub-centimeter nodule analysis; clinical deployment without addressing the domain gap between simulated and real radiographs [S1, unofficial].

## Background
AReT was proposed by Suja Palaniswamy et al. in 2026 (arXiv preprint) [S1, unofficial]. It addresses the limitations of existing NeRF adaptations for medical imaging, such as MedNeRF, which require dense views or rely on population priors that degrade patient-specific accuracy. The method identifies a previously unreported failure mode in TensoRF when applied to X-ray attenuation fields: the default density shift of $-10$ suppresses density gradients. By setting $\Delta = 0$ and incorporating thorax-specific regularization, AReT enables stable volumetric reconstruction from ultra-sparse views [S1, unofficial].

## Sources
[S1] Sparse-View Lung Nodule Volumetry from Digitally Reconstructed Radiographs via AReT: Anatomy-Regularized TensoRF, https://doi.org/10.48550/arxiv.2606.02639, fetched 2026-10-26 (unofficial)
