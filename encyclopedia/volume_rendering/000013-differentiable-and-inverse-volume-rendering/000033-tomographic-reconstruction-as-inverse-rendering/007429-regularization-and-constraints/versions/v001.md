# Regularization and Constraints


## Overview
This family encompasses methods that impose structural, physical, or statistical priors on the volume representation during tomographic reconstruction or inverse rendering to resolve ambiguities inherent in limited or noisy data. While the parent framework [Tomographic reconstruction as inverse rendering](45-tomographic-reconstruction-as-inverse-rendering.md) defines the differentiable forward projection $\mathcal{P}$ and the optimization loop, these methods introduce specific terms $R(x)$ into the loss function to guide the solution toward physically plausible or geometrically consistent results. They differ from standard iterative solvers by explicitly encoding domain knowledge—such as surface normal continuity, visual hull boundaries, or information-theoretic entropy—rather than relying solely on data fidelity and generic smoothness.

## Shared principles
All methods in this family extend the core optimization problem defined in the parent article by adding regularization terms to the loss function $\mathcal{L}$:

$$
\min_x \mathcal{L}_{data}(\mathcal{P}(x), p) + \sum_j \lambda_j R_j(x)
$$

where $\mathcal{L}_{data}$ measures the discrepancy between measured projections $p$ and rendered projections $\hat{p} = \mathcal{P}(x)$, and $R_j(x)$ are specific regularizers with associated weights $\lambda_j$. The gradient computation follows the chain rule provided in the parent article, but the regularizers $R_j(x)$ often require custom gradient implementations or auxiliary variables (e.g., surface normals, entropy distributions) that are differentiable with respect to the volume parameters $x$.

## Subtopics

### [Weighted Prior Based Reconstruction](17313-weighted-prior-based-reconstruction.md)
[Weighted Prior Based Reconstruction](17313-weighted-prior-based-reconstruction.md) uses spatially varying weights to moderate the influence of prior template volumes in longitudinal scans. Unlike standard prior-based methods that apply constant weights, this method computes a discrepancy-based weight matrix $W$ to suppress prior dominance in regions with new structures while preserving prior information in unchanged regions. It is chosen when reconstructing changes in a known scene using high-quality prior templates.

### [FreeNeRF](17644-freenerf.md)
[FreeNeRF](17644-freenerf.md) improves sparse-view NeRF rendering by applying frequency regularization to control visible input frequencies and penalizing high-density regions near the camera to reduce floating artifacts. It achieves state-of-the-art performance on sparse datasets with minimal architectural changes. It is chosen when training NeRFs with very few views (2–10) to prevent overfitting to high-frequency noise and floating geometry.

### [InfoNeRF](17849-infonerf.md)
[InfoNeRF](17849-infonerf.md) is an information-theoretic regularizer that minimizes reconstruction inconsistency by imposing an entropy constraint on the density distribution along rays and an information gain reduction loss between neighboring viewpoints. It addresses noise in few-shot settings and degenerate overfitting to redundant views. It is chosen when novel view synthesis suffers from artifacts due to insufficient viewpoints or when training data contains redundant, similar viewpoints.

### [Per-voxel Per-material Offset Regularization](22909-per-voxel-per-material-offset-regularization.md)
[Per-voxel Per-material Offset Regularization](22909-per-voxel-per-material-offset-regularization.md) introduces continuous, per-voxel, per-material offset parameters to absorb local density variations and measurement errors in discrete volume reconstruction. It solves the problem of overlapping attenuation ranges caused by fabrication artifacts (e.g., in 3D-printed phantoms), allowing crisp material boundaries despite local density variations. It is chosen for discrete CT reconstruction of objects with known materials but uncertain local attenuation coefficients.

### Normal Smoothness Constraint
An adaptive constraint that ensures continuity of estimated surface normals in flat regions while preserving sharp features by weighting the normal difference based on albedo similarity [S1]. It is used in implicit NeRF geometry reconstruction to reduce high-frequency noise on continuous surfaces without blurring structural boundaries.

**Definition:**
The normal smoothness loss $L_{normalSM}$ encourages adjacent surface normals to be similar, but weakens this constraint when local albedo differences indicate an edge [S1].

**Steps:**
1.  Sample a perturbed point by adding uniform noise in the normalized 3D space with magnitude 0.01 to generate the neighbor point for $N_{near}$ [S1].
2.  Compute the edge-aware weight mask $M_i$ from the albedo color difference $\tau = \| c_i - c_j \|_2$. If $\tau \le \tau_n = 0.15$, set $M_i = 1.0$; otherwise, set $M_i = 0.05$ [S1].

**Formula:**
$$
L_{normalSM} = \frac{1}{N} \sum_{i=1}^{N} M_i \cdot \| N_{pi} - N_{near} \|
$$
where $N_{pi}$ is the surface normal at point $p$, $N_{near}$ is the normal at a randomly perturbed neighboring point, and $M_i$ is the edge-aware weight mask [S1].

**Parameters:**
*   $\gamma_{sm}$: Weight for the normal smoothness term in the final objective function. Selected from $\{0.001, 0.005, 0.01\}$ [S1].

**Pitfalls:**
*   **Over-smoothing across edges:** Without the mask $M_i$, the constraint would undesirably smooth across structural breaks like roofs or walls. The mask weakens the constraint when local albedo differences indicate an edge [S1].

### Visual Hull Constraint
A hard constraint restricting density to the visual hull $H$ constructed from target views to avoid residual density and tomography artifacts in outer regions [S2]. It is used in fluid or object reconstruction to constrain the solution space to plausible shapes defined by the input views.

**Definition:**
The density $\rho$ is restricted to the visual hull $H$ using a Heaviside step function $H(\cdot)$, ensuring that no density exists outside the region visible from all target cameras [S2].

**Steps:**
1.  For single-view reconstruction, construct the hull using auxiliary image hulls created by rotating the original view around a symmetry axis of the physical setup [S2].
2.  Allow free motions via Neumann boundary conditions for velocity $u$ in inflow regions [S2].

**Formula:**
The visual hull $H_t$ is constructed by projecting the binary mask of the target image $I_{ct}$ back into the volume:
$$
H_t := \bigcap_{c \in C} R^{-1}(H(I_{ct} - \epsilon), c)
$$
where $R^{-1}$ is the back-projection operator, $H$ is the Heaviside step function, and $\epsilon$ is a threshold [S2]. The total loss includes a term that penalizes density outside $H_t$.

**Parameters:**
*   Not given in the sources.

**Pitfalls:**
*   **Single-view ambiguity:** The auxiliary hull from rotation is not physically correct but constrains the solution to plausible shapes, potentially introducing biases if the symmetry assumption is violated [S2].

## Background
Regularization and constraints in volume rendering and tomography have evolved from simple Tikhonov regularization to complex, learnable, and physically informed priors. Early methods focused on global smoothness (Total Variation) to handle noise [G]. With the advent of differentiable rendering and neural representations, regularization has become more granular, incorporating frequency domain constraints (FreeNeRF), information-theoretic measures (InfoNeRF), and physical constraints like visual hulls and transport equations [S1, S2]. The integration of prior templates with adaptive weighting allows for longitudinal analysis, while per-material offsets address real-world manufacturing imperfections [S1]. These methods are widely used in medical imaging, remote sensing (DSM reconstruction), and computational photography.

## Sources
[S1] Shading and Geometric Constraint Neural Radiance Field for DSM Reconstruction from Multi-View Satellite Images, https://doi.org/10.3390/rs18071091, fetched 2026-11-05
[S2] Global Transport for Fluid Reconstruction with Learned Self-Supervision, https://doi.org/10.1109/cvpr46437.2021.00168, fetched 2026-10-30
