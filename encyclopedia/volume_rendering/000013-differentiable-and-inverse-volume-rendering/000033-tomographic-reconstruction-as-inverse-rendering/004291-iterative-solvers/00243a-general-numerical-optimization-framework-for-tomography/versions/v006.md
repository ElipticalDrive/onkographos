# General Numerical Optimization Framework for Tomography

Also called: Regularized Inverse Problem Solver for Tomography, Regularized Optimization for Cryo-ET, Tomographic Reconstruction as Optimization, Variational Tomography Framework

## Overview
This framework formulates tomographic reconstruction as a regularized inverse problem, minimizing a data fidelity term plus a regularization term to handle noisy, angle-limited, or sparse projection data [S1][S2, unofficial]. It sets itself apart by decoupling the optimization into independent proximal operators, allowing the flexible incorporation of complex, non-smooth regularizers and physics-based priors [S1][S2, unofficial].

## Shared principles
The framework solves the inverse problem by minimizing an objective function composed of a data fidelity term and a regularization term [S2, unofficial]. The image formation model in log space is linear, derived from the Beer-Lambert Law [S1]. The measured intensity $I(p_i)$ for a pixel $p_i$ is related to the incident intensity $\hat{I}_0(p_i)$ and the attenuation cross-section (density) $\sigma$ along the ray $r_{p_i}(t)$:
$$ \hat{I}(p_i) = -\log \frac{I(p_i)}{\hat{I}_0(p_i)} = \int \sigma(r_{p_i}(t)) dt $$
Discretizing the volume into $N$ voxels and the projections into $M$ rays yields the linear system $W \mathbf{v} = \mathbf{p}$, where $\mathbf{v} \in \mathbb{R}^N$ is the unknown volume, $\mathbf{p} \in \mathbb{R}^M$ is the set of known ray integrals (tilt-series), and $W \in \mathbb{R}^{M \times N}$ is the weight matrix representing voxel contributions to ray integrals [S1].

The reconstruction is formulated as the optimization problem:
$$ \arg \min_{\mathbf{v}} \frac{1}{2} \|W \mathbf{v} - \mathbf{p}\|_2^2 + \Gamma(\mathbf{v}) $$
where the first term is the data fidelity (least-squares error) and the second term $\Gamma(\mathbf{v})$ is a non-linear regularizer [S1]. To apply proximal algorithms, the problem is rewritten in a constrained form:
$$ \arg \min_{\mathbf{v}, \mathbf{z}} F(\mathbf{v}) + G(\mathbf{z}) \quad \text{subject to} \quad K\mathbf{v} = \mathbf{z} $$
where $F(\mathbf{v}) = \frac{1}{2} \|W \mathbf{v} - \mathbf{p}\|_2^2$ is the data fidelity term, $G(\mathbf{z}) = \Gamma(\mathbf{v})$ is the regularization term, and $K$ is a linear operator related to the regularization term [S1].

The core mechanism is the decomposition of this problem into independent proximal operators: one for the linear least squares data term, and one for the non-linear regularizer [S2, unofficial]. Due to the large size of $W$, a matrix-free iterative solver is required to control memory consumption [S2, unofficial]. SART yields better results than Conjugate Gradients for solving the data term proximal operator, especially with sparse numbers of projections [S1]. The regularization proximal operator handles the non-smooth regularizer term and depends on the specific choice of $g$ (e.g., Total Variation) [S1].

This modular approach allows the framework to support 3D super-resolution, sparse view tomography, space-time reconstruction, warp-and-project deforming objects, and dynamic fluid capture [S2, unofficial].

## Subtopics

### [Chambolle-Pock Algorithm](3282-chambolle-pock-algorithm.md)
A first-order primal-dual optimization method that decouples the objective into two independent proximal sub-problems (data fidelity and regularization), allowing efficient handling of large-scale sparse systems. Choose when solving composite convex problems with non-smooth regularizers.

### [Coupled Density and Velocity Estimation](4622-coupled-density-and-velocity-estimation.md)
A physics-based strategy coupling density and velocity fields through transport and incompressibility priors. Choose when reconstructing fluid flow fields from sparse multi-view video data where standard tomography fails due to few cameras.

### [Stochastic Tomography](8886-stochastic-tomography.md)
A matrix-free, grid-free iterative solver using random walks and kernel density estimation. Choose when avoiding fixed spatial discretization is required, or for interactive reconstruction where explicit matrix inversion is infeasible.

### Generative Tomography Reconstruction via Optimization
Solves the linear inverse problem $A\mathbf{x}=\mathbf{y}$ by optimizing over a latent variable $\mathbf{z}$ of a generative model $G(\mathbf{r}, \mathbf{z})$ rather than directly over the image $\mathbf{x}$ [S3, unofficial]. It uses a weighted data fidelity term to account for non-uniform confidence in estimated sinogram values and a generator prior [S3, unofficial]. The optimization objective is:
$$ \arg\min_{\mathbf{z} \in S^m} \|A G(\mathbf{r}, \mathbf{z}) - \mu(\mathbf{r})\|_{\sigma(\mathbf{r})}^2 + \lambda D(G(\mathbf{r}, \mathbf{z})) $$
where $\mathbf{z}$ lies on an $m+1$ dimensional sphere, $\mu(\mathbf{r})$ is the estimated sinogram, and $\sigma(\mathbf{r})$ represents the confidence [S3, unofficial].

## Background
The framework originated from the need to reconstruct high-quality volumes from noisy and angle-limited tilt-series in cryogenic electron tomography (Cryo-ET) [S1]. It was formalized as a proximal algorithm framework to handle non-smooth, constrained, and large-scale optimization problems [S1]. The approach was subsequently generalized to X-ray CT for 3D super-resolution, sparse view reconstruction, and space-time reconstruction of deforming objects [S2, unofficial]. It has also been applied to dynamic fluid capture using RGB cameras [S2, unofficial].

## Sources
[S1] GPU Accelerated 3D Tomographic Reconstruction and Visualization From Noisy Electron Microscopy Tilt-Series., https://doi.org/10.1109/tvcg.2022.3230445, fetched 2026-10-28
[S2] Space-Time Tomographic Reconstruction of Deforming Objects, https://vccimaging.org/Publications/Zang2020Thesis/Zang2020Thesis.pdf, fetched 2026-10-30 (unofficial)
[S3] Generative Tomography Reconstruction, https://core.ac.uk/works/91148923, fetched 2026-10-31 (unofficial)
