# Laplacian Regularization

Also called: Laplacian Regularization for Sparse View Reconstruction

## Overview
Laplacian Regularization is a physics-based tomographic reconstruction technique that enforces smoothness on the spatial distribution of volume properties (such as extinction coefficients) by penalizing their Laplacian. It addresses the ill-posed nature of sparse-view reconstruction, particularly in smoke and spray imaging, where limited angular data leads to noisy or artifact-ridden results. It distinguishes itself from standard Tikhonov regularization by directly penalizing the second-order spatial derivatives (curvature) of the volume field, thereby preserving sharp features better while suppressing high-frequency noise [S1].

## How it works
**Inputs:**
- A set of $N$ training view images $I = \{I_i \in \mathbb{R}^M\}_{i=1}^N$.
- Associated camera poses $P = \{\phi_i \in \mathbb{R}^{3 \times 4}\}_{i=1}^N$.
- A neural radiance field parameterized by $\Theta$, denoted $F_\Theta(p, \theta) = (g \circ f(p, \theta), h \circ f(p))$, which outputs color $c$ and density $\sigma$ for a point $p$ and direction $\theta$ [S1].

**Outputs:**
- Optimized parameters $\Theta$ that minimize the total loss, including the Laplacian regularization term.

**Data Structures:**
- The volume is represented implicitly by the neural network $F_\Theta$. The regularization is applied to the output density field $\sigma_\Theta(x)$ defined over the domain $[-1, 1]^3$.

**Formulas:**
The Laplacian regularization term $R_{Lap}(\Theta)$ is defined as the integral of the absolute Laplacian of the density field over the volume:
$$ R_{Lap}(\Theta) = \int_{[-1,1]^3} |\Delta_x \sigma_\Theta(x)| dx $$
where $\Delta = \text{div} \cdot \nabla$ is the Laplacian operator [S1].

In practice, this integral is approximated by a discrete sum over a 3D grid of resolution $N \times N \times N$ with spacing $\delta$:
$$ R_{Lap}(\Theta) \approx \sum_{i=1}^N \sum_{j=1}^N \sum_{k=1}^N |\Delta_x \sigma_\Theta(\delta i, \delta j, \delta k)| \delta^3 $$
where $\sigma_\Theta(\delta i, \delta j, \delta k)$ is the density value at grid point $(\delta i, \delta j, \delta k)$ [S1].

**Steps:**
1. Initialize training view images $I$ and camera poses $P$.
2. Define the neural radiance field $F_\Theta$ mapping spatial coordinates and viewing directions to color and density [S1].
3. For each training iteration:
    a. Cast rays through the scene using camera poses $P$.
    b. Sample points along rays and query $F_\Theta$ to obtain density $\sigma_\Theta$ and color $c$.
    c. Render the view and compute the photometric loss against ground truth images $I$.
    d. Compute the Laplacian regularization term $R_{Lap}(\Theta)$ by evaluating the density field on a 3D grid and calculating the discrete Laplacian magnitude.
    e. Update $\Theta$ to minimize the combined loss (photometric + regularization) [S1].

**Parameters:**
- Grid resolution $N$: Not given in the sources.
- Grid spacing $\delta$: Not given in the sources.
- Regularization weight: Not given in the sources.

**Edge Cases and Pitfalls:**
- The discrete approximation assumes a uniform grid over $[-1, 1]^3$. Boundary conditions for the Laplacian calculation are not specified [S1].
- The absolute value $|\cdot|$ makes the term non-differentiable at zero, which may require subgradient methods or smoothing in implementation [G].

## Subtopics

## Operating characteristics
- **Time Complexity:** Depends on the number of grid points $N^3$ for regularization and the number of rays/samples for rendering. Not given in the sources.
- **Memory Complexity:** Requires storing the neural network parameters and potentially the 3D grid for regularization evaluation. Not given in the sources.
- **Strengths:** Improves continuity and generates smooth geometry representations, reducing severe noise compared to baselines [S1].
- **Limitations:** Relies on the assumption that the volume field is smooth; may oversmooth sharp discontinuities if the weight is too high. Not given in the sources.

## Background
Laplacian Regularization is employed in the context of Aug-NeRF, which enhances Neural Radiance Fields with physically-grounded augmentations. The method is cited in the context of improving the smoothness of depth maps and geometry representations in scenes generated with a physics-based renderer [S1].

## Sources
[S1] Aug-NeRF: Training Stronger Neural Radiance Fields with Triple-Level Physically-Grounded Augmentations, https://doi.org/10.1109/cvpr52688.2022.01476, fetched 2026-10-26
