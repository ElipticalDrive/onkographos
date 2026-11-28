# Volumetric Supervision for NeRF

Also called: Hybrid NeRF reconstruction, NeRF with volumetric loss

## Overview
Volumetric Supervision for NeRF is a hybrid reconstruction method designed to improve the fidelity of the canonical volume in dynamic tomography by combining projection-based training with volumetric supervision from high-fidelity static X-CT data at key time steps ($t=0$ and $t=1$) [S1, unofficial]. It addresses the observation that standard neural rendering methods often fail to achieve high normalized correlation coefficients for complex objects, whereas conventional data-rich X-CT can reconstruct such volumes with high fidelity [S1, unofficial]. It sets itself apart from standard NeRF optimization by explicitly incorporating a ground-truth volumetric loss term derived from external high-resolution scans, rather than relying solely on 2D projection consistency [S1, unofficial].

## How it works
**Inputs and Outputs:**
*   **Inputs:** Sparse 2D projections for intermediate time steps (e.g., 2 projections for $t=0.1, \dots, 0.9$) [S1, unofficial]. High-fidelity 3D X-CT volumes for key time steps ($t=0$ and $t=1$) [S1, unofficial]. A set of 2D projections for the canonical step (e.g., 256 projections for $t=0$) [S1, unofficial]. A deformation field parametrization (e.g., B-spline grid) [S1, unofficial].
*   **Outputs:** A high-fidelity canonical volume at $t=0$ (and optionally $t=1$) [S1, unofficial]. A trained deformation field mapping the canonical volume to dynamic states [S1, unofficial].

**Data Structures:**
*   **Canonical Volume:** Represented as a 3D density field $\sigma(\mathbf{x})$ evaluated by a neural network or grid structure [S1, unofficial].
*   **Ground Truth X-CT:** A 3D volume of density values $\sigma_{gt}(\mathbf{x})$ obtained from high-fidelity static scans [S1, unofficial].
*   **Deformation Field:** Parametrized by a B-spline grid with a cited resolution of $6 \times 6 \times 6$ [S1, unofficial].

**Algorithm Steps:**
1. Initialize the neural radiance field for the canonical volume and the deformation field (e.g., B-spline grid with resolution $6 \times 6 \times 6$) [S1, unofficial].
2. Initialize the volumetric supervision weight $\lambda = 0$ [S1, unofficial].
3. For the first 200 iterations, render the canonical volume from the 256 projections at $t=0$, compute $L_{proj}$, and optimize network parameters using $L = L_{proj}$ [S1, unofficial].
4. Set $\lambda = 0.005$ [S1, unofficial].
5. For subsequent iterations, render the canonical volume from the 256 projections, compute $L_{proj}$, compute $L_{vol}$ using the normalized correlation between the predicted canonical volume density and the high-fidelity X-CT ground truth at $t=0$, compute total loss $L = L_{proj} + 0.005 L_{vol}$, and optimize network parameters [S1, unofficial].
6. Using the reconstructed canonical volume, train the deformation field to map to intermediate time steps ($t=0.1, \dots, 0.9$) using the sparse 2 projections per step [S1, unofficial].

**Formulas:**
The total training loss $L$ for the canonical volume reconstruction is defined as:
$$L = L_{proj} + \lambda L_{vol}$$
Where:
*   $L_{proj}$ is the projection loss, computed by rendering the canonical volume from the available 2D projections and comparing against measured projection data [S1, unofficial].
*   $L_{vol}$ is the volumetric loss, calculated as the negative normalized correlation $C_{cor}$ between the predicted density of the canonical volume and the ground truth X-CT volume [S1, unofficial].
*   $\lambda$ is a weighting coefficient for the volumetric supervision [S1, unofficial].
*   $C_{cor}$ denotes the normalized correlation coefficient between the predicted density field and the ground truth X-CT volume [S1, unofficial].

**Parameters:**
*   $\lambda$: Volumetric supervision weight. Set to 0 for the first 200 iterations, then set to 0.005 [S1, unofficial].
*   B-spline Grid Resolution: $6 \times 6 \times 6$ [S1, unofficial].
*   Projections at $t=0$: 256 [S1, unofficial].
*   Projections at intermediate steps: 2 [S1, unofficial].

**Edge Cases and Pitfalls:**
*   **Initialization:** The coefficient $\lambda$ must be 0 initially to allow the projection-based loss to establish a basic geometry before introducing the volumetric constraint [S1, unofficial].
*   **Data Availability:** This method requires high-fidelity static X-CT data at key time steps, which may not be available in all dynamic tomography scenarios [S1, unofficial].

## Operating characteristics
*   **Accuracy:** Volumetric supervision significantly improves reconstruction quality, improving both normalized correlation and Peak Signal-to-Noise Ratio (PSNR) on simulated lattice datasets compared to training purely with projections [S1, unofficial].
*   **Displacement Error:** In simulated deformation experiments (Kelvin lattice), the displacement error $E_{disp}$ was $0.5 \times 10^{-5}$ when using both volumetric supervision and rich data at $t=1$, compared to $6.4 \times 10^{-5}$ with neither [S1, unofficial].
*   **Complexity:** The method adds the computational cost of computing the normalized correlation between the predicted volume and the ground truth X-CT volume to the training loop [S1, unofficial].
*   **Strengths:** Synergistically improves accuracy when combined with rich data at multiple time steps [S1, unofficial].
*   **Limitations:** Requires high-fidelity X-CT data at key time steps, which increases data acquisition requirements [S1, unofficial].

## Background
This method was proposed in the context of dynamic tomography, where neural rendering is used to reconstruct 4D volumes from sparse 2D projections [S1, unofficial]. It leverages the observation that while neural rendering struggles with high normalized correlation for complex objects, conventional X-CT can achieve high fidelity when many projections are available [S1, unofficial]. The method was tested on simulated lattice datasets and deformation experiments inspired by uniaxial tensile tests [S1, unofficial].

## Sources
[S1] Neural rendering enables dynamic tomography, https://doi.org/10.48550/arxiv.2410.20558, fetched 2026-10-26 (unofficial)
