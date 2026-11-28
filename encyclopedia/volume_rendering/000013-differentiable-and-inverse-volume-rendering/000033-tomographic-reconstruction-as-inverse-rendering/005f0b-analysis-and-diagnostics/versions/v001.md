# Analysis and Diagnostics
## Overview
Analysis and Diagnostics methods evaluate the quality, stability, and information content of tomographic reconstructions without necessarily updating the volume parameters. They solve the problem of verifying reconstruction success and diagnosing artifacts by analyzing the relationship between the measured sinogram and the reconstructed volume. Unlike iterative solvers that minimize a loss function to find the best volume estimate, these methods provide diagnostic metrics (such as SNR or error volumes) or algebraic characterizations (such as null space analysis) to assess the reliability of the reconstruction process .

## Shared principles
All methods in this family rely on the differentiability of the forward projection operator $\mathcal{P}$ and its adjoint (back-projection) $\mathcal{P}^T$ [G]. The core optimization problem is generally formulated as minimizing a loss function $\mathcal{L}$ between the measured sinogram $p$ and the rendered projection $\hat{p} = \mathcal{P}(x)$ of the current volume estimate $x$:

$$
\min_x \mathcal{L}(\mathcal{P}(x), p) + \lambda R(x)
$$

where $R(x)$ is a regularizer (e.g., Total Variation, neural prior) and $\lambda$ is a weighting factor .

The gradient of the loss with respect to the volume $x$ is computed via the chain rule:

$$
\frac{\partial \mathcal{L}}{\partial x} = \mathcal{P}^T \left( \frac{\partial \mathcal{L}}{\partial \hat{p}} \right)
$$

The term $\frac{\partial \mathcal{L}}{\partial \hat{p}}$ is the residual gradient in projection space (e.g., $p - \hat{p}$ for MSE), and $\mathcal{P}^T$ is the back-projection operator [G].

Key shared components include:
1.  **Forward Projection:** Ray marching through the volume representation (voxels, octrees, or neural fields) to compute line integrals of density/absorption coefficients .
2.  **Backward Projection (Adjoint):** Distributing the error from the 2D detector plane back into the 3D volume. This is mathematically the transpose of the forward operator, ensuring stable convergence in iterative methods [G].
3.  **Optimization Loop:** An iterative process (e.g., SGD, Adam) that updates the volume parameters (voxel values, neural network weights, or Gaussian parameters) to reduce the projection error .

## Subtopics

### [Explicit Radon Transform Analysis](9108-explicit-radon-transform-analysis.md)
Explicit Radon Transform Analysis constructs and analyzes the explicit forward projection matrix $A$ for a given scanning geometry to diagnose the ill-posedness of tomographic reconstruction. It identifies the singular values and the null space of the operator, revealing volumetric perturbations that are invisible to the measurements. Unlike iterative solvers that approximate the inverse, this method provides a direct algebraic characterization of the information loss in sparse or limited-angle scenarios.

### [Optimal Projection Angle Selection via Mutual Information Minimization](11067-optimal-projection-angle-selection-via-mutual-information-mi.md)
This method selects the optimal set of projection angles $\theta_i$ for tomographic reconstruction by minimizing the posterior entropy of the inferred 3D volume given the projections, $H(\mathcal{D}|\mathcal{X})$. It operates on the principle that maximizing the joint entropy of the projections $H(\mathcal{X})$ reduces the uncertainty of the underlying data, which is achieved by minimizing the sum of pairwise mutual information (MI) terms between all projection pairs. Unlike sibling methods that focus on reconstruction speed or specific geometric approximations, this approach is information-theoretic, deriving angle configurations that maximize data diversity based on statistical assumptions about the volume and noise. It is distinct from [Compressive Volume Rendering Recovery Methods](1915-compressive-volume-rendering-recovery-methods.md) by explicitly modeling the correlation structure between projections via Gaussian priors, and from [FastFDK](3315-fastfdk.md) by optimizing the acquisition geometry rather than the reconstruction kernel.

### [Signal-to-Noise Ratio (SNR) for Tomography](14356-signal-to-noise-ratio-snr-for-tomography.md)
Signal-to-Noise Ratio (SNR) for Tomography is a metric used to evaluate the robustness of a tomographic reconstruction method against acquisition noise. It quantifies how well a reconstruction algorithm preserves signal integrity when the input sinogram is corrupted by noise, distinguishing between noise introduced by the acquisition process and noise induced by the reconstruction algorithm itself. Unlike standard image quality metrics that compare a reconstruction to a ground truth, SNR for Tomography compares two reconstructions: one from a clean sinogram and one from a noisy sinogram. This sets it apart from metrics like SSIM or MSE which measure fidelity to an ideal target, as SNR specifically isolates the method's sensitivity to input perturbations.

### [Error Volume Analysis](15301-error-volume-analysis.md)
Error Volume Analysis is a diagnostic technique for tomographic reconstruction that visualizes the residual error between the measured projection data (tilt-series) and the synthetic projections generated from the current volume estimate. It solves the problem of verifying reconstruction success by providing a direct consistency check between the reconstructed tomogram and the raw input data. Unlike standard reconstruction metrics that provide scalar values, this method produces a 3D error volume or 2D error projections, allowing users to assess the reliability and trustworthiness of the reconstruction by inspecting spatial error distribution. It is distinct from sibling methods like [Back-projection](2450-back-projection.md) or [Iterative Solvers](10781-iterative-solvers.md) because it does not update the volume parameters but rather analyzes the output of the reconstruction process to detect artifacts, misalignments, or insufficient regularization.

## Background
The family of Analysis and Diagnostics methods in tomographic reconstruction emerged from the need to quantify the reliability of inverse solutions, particularly in limited-angle or sparse-view scenarios where standard reconstruction algorithms may produce ambiguous or artifact-ridden results. Early work focused on algebraic approaches, such as the analysis of the Radon transform's null space, to understand the fundamental limits of information recovery . As computational power increased, information-theoretic approaches, such as mutual information minimization for angle selection, were developed to optimize acquisition protocols . More recently, with the rise of differentiable rendering and inverse rendering, diagnostic tools like Error Volume Analysis have become integral to verifying the consistency of learned volumetric representations against raw sensor data . These methods are widely used in electron tomography, medical CT, and computational imaging to validate reconstruction pipelines and guide experimental design.

## Sources
[S1] Signal-to-Noise Ratio (SNR) for Tomography, https://example.com/snr-tomography, 2023-10-01
[S2] Explicit Radon Transform Analysis, https://example.com/radon-analysis, 2023-10-01
