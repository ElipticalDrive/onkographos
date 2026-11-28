# Conical Reflected Features

Also called: Directional Sampling, Reflection Feature Downweighting, vMF Directional Sampling

## Overview
Conical Reflected Features estimates the expected volume-rendered feature over a von Mises-Fisher (vMF) distribution of reflected rays to decode into reflected color, avoiding the prohibitive cost of Monte Carlo integration [S1]. It differs from standard NeRF-Casting by explicitly modeling the angular spread of roughness via a conical sampling pattern and adapting Zip-NeRF's feature downweighting to the directional domain to prevent aliasing in distant reflections [S1].

## How it works
**Inputs:**
- Reflection direction $\mathbf{d}'$ (unit vector).
- Accumulated roughness $\bar{\kappa}$ (scalar).
- Number of sampled rays $K$.
- Number of sample points per ray $N'$.
- Position $\mathbf{x}$ and feature $\mathbf{f}(\mathbf{x})$ from the neural field.
- NGP grid resolution scale vector $\nu$.
- Cone scale parameter $\gamma$ (default 16).

**Outputs:**
- Downweighted feature vector $\bar{f}$ for the reflected cone.

**Data Structures:**
- No specific persistent data structures; computation is per-ray/cone.

**Steps:**

1.  **Compute the cone scale $\sigma(\mathbf{x})$** at the evaluation point $\mathbf{x}$ to determine the appropriate downweighting factor.
    a. Compute the directional Jacobian determinant of the contraction function $C$:
       $$ \det \mathbf{J}_C^{\text{dir}}(\mathbf{x}) = \left( \frac{2 \max(1, \|\mathbf{x}\|) - 1}{\max(1, \|\mathbf{x}\|^2)} \right)^2 $$
    b. Compute the cone scale:
       $$ \sigma(\mathbf{x}) = \gamma \cdot (\dot{r} + \bar{\rho}) \|\mathbf{x} - \mathbf{o}'\| \sqrt{\det \mathbf{J}_C^{\text{dir}}(\mathbf{x})} $$
       where $\dot{r}$ is the accumulated roughness from the surface interaction, $\bar{\rho}$ is the accumulated roughness from previous bounces, and $\mathbf{o}'$ is the origin of the reflected ray.

2.  **Apply Reflection Feature Downweighting** to the raw feature $\mathbf{f}(\mathbf{x})$ to prevent aliasing for high roughness or distant points:
    $$ \mathbf{f}_{\text{aa}}(\mathbf{x}) = \text{erf}\left(\left(\sqrt{8}\nu\sigma(\mathbf{x})\right)^{-1}\right) \odot \mathbf{f}(\mathbf{x}) $$
    where $\odot$ is element-wise multiplication. Note: $\nu$ is a vector of the same dimension as $\mathbf{f}$ containing the scale of the NGP grid resolution.

3.  **Generate $K$ sample directions** $\{\mathbf{d}'_1, \dots, \mathbf{d}'_K\}$ on the cone around $\mathbf{d}'$.
    a. Determine the angle $\psi$ between the mean direction $\mathbf{d}'$ and the offset reflections:
       $$ \cos(\psi) = \frac{K(\coth(\bar{\kappa}) - 1/\bar{\kappa}) - 1}{K - 1} $$
    b. Construct an orthonormal basis $(\mathbf{t}_1, \mathbf{t}_2, \mathbf{d}')$.
       i. Select a reference vector $\mathbf{u}$ for numerical stability:
          $$ \mathbf{u} = \begin{cases} \hat{\mathbf{z}} & \text{if } \|\mathbf{d}' - \hat{\mathbf{z}}\| < 0.9 \\ \hat{\mathbf{y}} & \text{otherwise} \end{cases} $$
       ii. Compute tangent vectors:
          $$ \mathbf{t}_1 = \frac{\mathbf{u} \times \mathbf{d}'}{\|\mathbf{u} \times \mathbf{d}'\|}, \quad \mathbf{t}_2 = \mathbf{d}' \times \mathbf{t}_1 $$
       iii. *Optimization Note:* During training, randomly rotate $\mathbf{t}_1$ and $\mathbf{t}_2$ in the tangent plane to break rotational symmetry. During evaluation, use the deterministic basis defined above.
    c. Compute the directions:
       $$ \mathbf{d}'_1 = \mathbf{d}' $$
       $$ \mathbf{d}'_{j+2} = \cos(\psi)\mathbf{d}' + \sin(\psi) \left( \cos\left(\frac{2j\pi}{K-1}\right) \mathbf{t}_1 + \sin\left(\frac{2j\pi}{K-1}\right) \mathbf{t}_2 \right) \quad \text{for } j \in \{0, \dots, K-2\} $$

4.  **Volume Render Features** for each sample direction $\mathbf{d}'_j$.
    For each $j \in \{1, \dots, K\}$:
    a. Sample $N'$ points $\mathbf{x}_j^{-1}$ along the ray starting at $\mathbf{o}'$ with direction $\mathbf{d}'_j$.
    b. Evaluate the downweighted features $\mathbf{f}_{\text{aa}}(\mathbf{x}_j^{-1})$ and densities at these points.
    c. Compute the volume-rendered feature for this ray:
       $$ \bar{f}(\mathbf{d}'_j) = \sum_{i=1}^{N'} w_j^{-1} \mathbf{f}_{\text{aa}}(\mathbf{x}_j^{-1}) $$
       where $w_j^{-1}$ are the standard volume rendering weights.

5.  **Average** the $K$ features to yield the final reflected feature:
    $$ \bar{f} = \frac{1}{K} \sum_{j=1}^K \bar{f}(\mathbf{d}'_j) $$

**Parameters:**
- $\gamma$: Fixed scale multiplier for cone scale computation. Typical value: 16 [S1].
- $K$: Number of sampled rays. Not given in the sources.
- $N'$: Number of sample points per ray. Not given in the sources.

**Edge Cases and Pitfalls:**
- **Distant Aliasing:** Using Zip-NeRF's original downweighting (geometric mean of 3D Jacobian eigenvalues) causes $\sigma(\mathbf{x})$ to approach 0 for points far from the origin due to strong contraction, resulting in significant aliasing in reflections of distant content [S1]. The directional Jacobian determinant prevents this by approaching a constant [S1].
- **Numerical Stability:** The denominator when normalizing $\mathbf{t}_1$ is bounded from below, ensuring stability for any reflection direction $\mathbf{d}'$ [S1].
- **Roughness Zero:** When accumulated roughness $\bar{\kappa}$ is zero, the cone collapses to a single ray, and $\mathbf{o}'$ is perfectly mirrored [S1].

## Operating characteristics
- **Time Complexity:** $O(K \cdot N')$ per reflection, where $K$ is the number of cone samples and $N'$ is the number of volume rendering samples per ray. This is significantly faster than full Monte Carlo integration which would require many more samples for low noise [S1].
- **Memory Complexity:** $O(1)$ additional memory per ray evaluation, as features are computed on the fly.
- **Strengths:** Efficiently approximates the integral of reflected features over a vMF distribution. Prevents aliasing in reflections of distant content by using a directional Jacobian determinant that does not vanish under contraction [S1].
- **Limitations:** Requires tuning of $K$ and $\gamma$. The deterministic basis selection during evaluation may introduce slight artifacts if not randomized during training, though the method claims stability [S1].
- **When to use:** When rendering reflections in large scenes with contraction (e.g., mip-NeRF 360/Zip-NeRF settings) where distant reflections would otherwise alias.
- **When not to use:** For perfectly smooth (specular) surfaces where a single ray cast is sufficient, though the method degrades gracefully to a single ray as $\bar{\kappa} \to 0$ [S1].

## Background
Conical Reflected Features was introduced in "NeRF-Casting: Improved View-Dependent Appearance with Consistent Reflections" (2024) [S1]. It builds upon the Zip-NeRF training procedure and the NeRF-Casting reflection model. The method addresses the limitation of standard NeRF-Casting in large-scale scenes with spatial contraction, where traditional feature downweighting fails for distant objects [S1].

## Sources
[S1] NeRF-Casting: Improved View-Dependent Appearance with Consistent Reflections, https://arxiv.org/abs/2405.14871v1, fetched 2026-10-26
