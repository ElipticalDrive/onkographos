# Regularization and Gradient Losses

Also called: SDF Regularization, Gradient Constraints, Surface Consistency Losses

## Overview
This family defines auxiliary loss terms used during the training of neural Signed Distance Field (SDF) networks to enforce geometric priors, improve surface smoothness, and prevent artifacts in regions lacking direct supervision. Unlike the primary Eikonal loss which enforces unit gradient magnitude everywhere, these terms target specific geometric properties such as surface alignment between different field representations, local smoothness of the distance function, and the suppression of "free surfaces" in unobserved volumes. They are typically combined with photometric or depth supervision to stabilize the optimization of implicit surfaces.

## Shared principles
All methods in this family operate on the output of a neural network $f(\mathbf{x})$ that predicts the SDF value $s$ at a query point $\mathbf{x} \in \mathbb{R}^3$. The core mechanism involves computing a scalar loss $\mathcal{L}$ based on the predicted SDF values $s(\mathbf{x})$ or their spatial gradients $\nabla s(\mathbf{x})$, and backpropagating this loss to update the network weights.

Common characteristics include:
1.  **Query Points**: Losses are computed over a set of sampled points $\mathcal{Q}$, which may include points on the surface (where $s \approx 0$), points in free space ($s > 0$), or points inside the object ($s < 0$).
2.  **Gradient Dependence**: Many terms rely on the gradient of the SDF, $\nabla s(\mathbf{x})$, either explicitly (e.g., smoothness between adjacent gradients) or implicitly (e.g., enforcing magnitude via Eikonal, though Eikonal is covered in the parent article).
3.  **Regularization Goal**: The objective is not to fit observed data directly but to constrain the solution space to be geometrically plausible (smooth, compact, consistent).
4.  **Weighting**: Each loss term is typically multiplied by a hyperparameter $\lambda$ to balance its influence against other losses (e.g., photometric loss, Eikonal loss).

## Subtopics

### [Surface SDF Loss](14235-surface-sdf-loss.md)
Enforces geometric consistency between a primary SDF network and a secondary Directional Distance Field (DDF) network. It ensures that points predicted by the DDF to lie on the surface actually correspond to the zero-level set of the SDF. This is particularly useful when training multi-representation models to resolve depth ambiguities. It differs from standard SDF supervision by comparing two learned representations rather than a network against ground truth. See [Surface SDF Loss](14235-surface-sdf-loss.md) for implementation details.

### SDF Smoothness Loss
Regularizes the reconstructed surface by penalizing the difference between gradients of the SDF at adjacent sampled points. This encourages geometric smoothness and reduces high-frequency noise in the predicted distance field. It is computed by comparing the gradient $\nabla d_\Omega(\mathbf{p})$ at a point $\mathbf{p}$ with the gradient at a perturbed point $\mathbf{p} + \varepsilon$. The loss is defined as [S2]:
$$
\mathcal{L}_s = \frac{1}{N_{sm}} \sum_{\mathbf{p} \in \mathcal{X}} \| \nabla d_\Omega(\mathbf{p}) - \nabla d_\Omega(\mathbf{p} + \varepsilon) \|_2
$$
where $\mathcal{X}$ is the set of sampled points, $N_{sm}$ is the number of points used for computing the loss, $d_\Omega$ is the SDF output, and $\varepsilon$ is a small perturbation vector. This term is applied to the geometry MLP parameters.

### Sparse Points SDF Loss
A variant used in sparse-view reconstruction pipelines (e.g., SDFStudio) that applies SDF supervision only to a subset of points extracted from sparse input data. The process involves:
1.  Extracting sparse points and weights from the input batch.
2.  Filtering points to keep only those inside the unit sphere (norm < 1.0).
3.  Evaluating the SDF network at these filtered points.
4.  Computing the loss as the mean of the absolute SDF values weighted by the input weights.
In the specific implementation cited, the loss is multiplied by a factor of 0.0, effectively disabling it or serving as a placeholder for future tuning [S1, unofficial]. This suggests that in some configurations, sparse point supervision is optional or handled differently.

### SparseNeuS Sparseness Regularization
Designed to prevent the generation of "uncontrollable free surfaces" behind visible surfaces in sparse-view neural surface reconstruction. It penalizes points that are close to the zero-level set in unobserved regions, encouraging the SDF values to be large (far from zero) in free space behind the visible geometry. The loss is defined as [S3]:
$$
\mathcal{L}_{sparse} = \frac{1}{\|\mathbb{Q}\|} \sum_{q \in \mathbb{Q}} \exp(-\tau |s(q)|)
$$
where $\mathbb{Q}$ is the set of sampled points, $s(q)$ is the predicted SDF value, and $\tau$ is a temperature parameter controlling the penalty strength. By minimizing this term, the network is encouraged to make $|s(q)|$ large for points $q$ behind visible surfaces, thus pushing the zero-level set away from these ambiguous regions.

### SDF Rescaling Term for Points Behind Visible Surfaces
A complementary regularization term used in SparseNeuS to further suppress free surfaces. It directly penalizes points located behind visible surfaces by encouraging their absolute SDF values to be large. The term is defined as [S3]:
$$
\mathcal{L}_{rescale} = |s(q)| \cdot \tau
$$
where $|s(q)|$ is the absolute SDF value of a sampled point $q$ behind the visible surface, and $\tau$ is a hyperparameter. This linear penalty works in conjunction with the exponential sparseness term to ensure compact geometry.

## Background
Regularization of neural SDFs has become critical as implicit representations are applied to sparse-view and unstructured data where photometric supervision is insufficient. The Eikonal loss [12] (covered in the parent article) established the baseline for enforcing unit gradients. Subsequent works introduced specialized terms to handle specific failure modes:
*   **Smoothness**: Early neural surface reconstruction methods noted that raw SDF predictions can be noisy, leading to jagged meshes. Gradient-based smoothness losses were introduced to enforce local continuity [S2].
*   **Multi-Representation Consistency**: As models like MonoSDF and others began using directional distances to resolve depth ambiguities, losses enforcing consistency between different field representations (SDF vs. DDF) became necessary [S2].
*   **Sparse View Artifacts**: In sparse-view settings (e.g., SparseNeuS), the lack of supervision behind visible surfaces leads to "flying artifacts" or free surfaces. Sparseness regularization and rescaling terms were proposed to penalize zero-crossings in these unobserved regions, ensuring the reconstructed geometry is compact and well-defined [S3].

## Sources
[S1] autonomousvision/sdfstudio, https://github.com/autonomousvision/sdfstudio, fetched 2026-10-26 (unofficial)
[S2] Ray-Distance Volume Rendering for Neural Scene Reconstruction, https://doi.org/10.1007/978-3-031-72630-9_22, fetched 2026-10-26
[S3] SparseNeuS: Fast Generalizable Neural Surface Reconstruction from Sparse Views, https://doi.org/10.1007/978-3-031-19824-3_13, fetched 2026-10-26
