# Reparameterization Estimators

Also called: Reparameterization Trick, Change of Variables in Differentiable Rendering

## Overview
Reparameterization Estimators are a family of differentiable rendering techniques that compute unbiased gradients by transforming parameter-dependent random variables (such as sample locations or ray directions) into parameter-independent ones, allowing the derivative to pass through the sampling distribution [S1, S2, S3]. Unlike [Automatic Differentiation (Reverse/Adjoint Mode)](7151-automatic-differentiation-reverse-adjoint-mode.md), which differentiates the computational graph directly and suffers from high memory usage and discontinuous gradients at visibility boundaries, reparameterization methods ensure that sample points are explicit, differentiable functions of the scene parameters, enabling gradient flow through geometric and density discontinuities [S1, S2, S3]. They set themselves apart from [Analytical Gradients](7613-analytical-gradients.md) by relying on Monte Carlo estimation of the transformed integrand rather than closed-form inversion of the rendering integral, making them applicable to complex path tracing scenarios [S1].

## Shared principles
The core mechanism relies on the **reparameterization trick** (or change of variables). Let the rendering integral be $I(\theta) = \int_{\Omega_\theta} f(x, \theta) dx$, where the domain $\Omega_\theta$ or the sampling distribution $p(x|\theta)$ depends on parameters $\theta$. Direct differentiation yields boundary terms that are difficult to estimate. Instead, a transformation $x = T(u, \theta)$ is introduced, where $u$ is a parameter-independent random variable (e.g., $u \sim U[0,1]$ or $u \sim \text{Uniform}(S^2)$) [S1, S2, S3].

The integral becomes:
$$
I(\theta) = \int_{\Omega_u} f(T(u, \theta), \theta) \left| \det \frac{\partial T}{\partial u} \right| du
$$
Since the domain $\Omega_u$ and the distribution of $u$ are independent of $\theta$, the gradient can be moved inside the integral:
$$
\nabla_\theta I(\theta) = \int_{\Omega_u} \nabla_\theta \left[ f(T(u, \theta), \theta) \left| \det \frac{\partial T}{\partial u} \right| \right] du
$$
This estimator is unbiased provided $T$ is differentiable with respect to $\theta$ and $u$ [S1, S2, S3]. The Jacobian determinant $\left| \det \frac{\partial T}{\partial u} \right|$ accounts for the change in volume element. In practice, this requires:
1.  **Mapping Construction:** Defining $T$ such that it maps fixed samples to parameter-dependent locations (e.g., inverse CDF for density, rotation for silhouette motion) [S1, S2, S3].
2.  **Jacobian Evaluation:** Computing or approximating the determinant of the transformation's Jacobian [S1, S3].
3.  **Gradient Accumulation:** Summing the gradients of the transformed integrand over Monte Carlo samples [S1, S2, S3].

## Subtopics

### [Reparameterized Volume Sampling](3309-reparameterized-volume-sampling.md)
Reparameterized Volume Sampling computes color estimates by sampling points on a ray using the inverse of the cumulative distribution function (CDF) of the density field. It solves the problem of non-differentiable sampling in volume rendering by ensuring that sample locations are explicit, differentiable functions of the density parameters, allowing gradients to back-propagate through the sampling locations. Unlike NeRF's hierarchical sampling which uses an auxiliary coarse model trained separately, this method learns sampling points end-to-end without auxiliary networks. Compared to standard importance sampling with uniform distribution, this stratified reparameterized sampling exhibits lower variance.

### [Boundary Path Integral Formulation](3389-boundary-path-integral-formulation.md)
The Boundary Path Integral Formulation is a method for estimating the derivatives of the path integral in differentiable volume rendering with respect to object shape parameters. It solves the problem of computing gradients when scene geometry depends on parameters $\theta$ by separating the differentiation of the path integral into interior and boundary terms. Unlike its siblings [Reparameterized Volume Sampling](3309-reparameterized-volume-sampling.md) and [Reparameterized Monte Carlo Volume Rendering](4798-reparameterized-monte-carlo-volume-rendering.md), which typically rely on differentiable sampling within the volume or standard CDF inversion, this method converts interior volume integrals into boundary area integrals using the divergence theorem. This eliminates the need to evaluate the deformation mapping inside the volume, avoiding expensive tetrahedralizations.

### [Reparameterized Monte Carlo Volume Rendering](4798-reparameterized-monte-carlo-volume-rendering.md)
Reparameterized Monte Carlo Volume Rendering (R/MC) is a differentiable volume rendering method that estimates expected radiance by sampling from the density-induced probability distribution using the reparameterization trick. It solves the problem of computing gradients for volume rendering integrals where the domain of integration (the ray segment) and the sampling distribution depend on the scene parameters (density field). Unlike standard Monte Carlo volume rendering, which suffers from discontinuous gradients when samples cross object boundaries, R/MC maps parameter-independent uniform random variables to ray distances via the inverse opacity function, ensuring the gradient estimator remains unbiased and differentiable. It sets itself apart from [Reparameterized Volume Sampling](3309-reparameterized-volume-sampling.md) by explicitly normalizing the integration domain to opacity space $[0, y_f]$ and solving for ray distances through inverse CDF approximation, whereas [Boundary Path Integral Formulation](3389-boundary-path-integral-formulation.md) handles boundaries via path integral formulations.

### [Differential path integral estimator with MIS and NEE](6332-differential-path-integral-estimator-with-mis-and-nee.md)
This method computes the gradient of a rendered image with respect to scene parameters $\theta$ by integrating differential measurement contributions over path space, using Multiple Importance Sampling (MIS) to combine BRDF and emitter sampling, and Next Event Estimation (NEE) to connect paths to emitters. It solves the problem of high variance in gradient estimation for scenes with varying material properties (e.g., roughness) by allowing the "differential vertex" (the vertex where the parameter derivative is applied) to be located at any intermediate vertex or at the emitter connection. Unlike standard NEE, which only connects paths to emitters at the final vertex, this method combines BRDF and emitter sampling probabilities into a single mixture weight, significantly reducing variance for microfacet and mixture BRDFs. It builds on the reparameterization principles of [Reparameterization Estimators](6499-reparameterization-estimators.md).

### [Reparametrized Integral Gradients](6785-reparametrized-integral-gradients.md)
Reparametrized Integral Gradients is a technique for differentiable rendering that eliminates bias in gradients caused by moving visibility discontinuities by applying changes of variables to render the discontinuities static in the transformed domain. Unlike explicit edge sampling methods that require detecting and sampling geometric boundaries, this method reparameterizes the integration domain (such as spherical domains for light transport) so that the discontinuity points become independent of the scene parameters $\pi$. This allows the gradient to be computed via standard Monte Carlo integration over the interior, avoiding the boundary terms that arise when differentiating under the integral sign. It is distinct from [Reparameterized Volume Sampling](3309-reparameterized-volume-sampling.md) which focuses on density fields, and [Boundary Path Integral Formulation](3389-boundary-path-integral-formulation.md) which explicitly separates and samples boundary terms.

### [Plateau-Reduced Differentiable Path Tracing](8826-plateau-reduced-differentiable-path-tracing.md)
Plateau-Reduced Differentiable Path Tracing (PR-DPT) is an inverse rendering method that alleviates optimization plateaus (regions of zero gradient) by convolving the rendering equation with a smoothing kernel over the parameter space. Unlike standard differentiable renderers that suffer from discontinuous gradients due to visibility changes, PR-DPT creates a smooth objective function, allowing gradient-based optimizers to converge on problems with intricate light transport like caustics. It sets itself apart from sibling reparameterization methods by not requiring a special integrator for visibility-induced discontinuities or the construction of a gradient tape, instead relying on stochastic sampling of a blurred parameter space.

### [Monte Carlo Estimation of Differentiable Path Integrals](10814-monte-carlo-estimation-of-differentiable-path-integrals.md)
This method estimates the gradient of a path integral rendering equation with respect to scene parameters $\pi$ by decomposing the derivative into interior contributions (derivative of the path contribution function) and boundary contributions (derivative of the visibility boundary). It distinguishes itself from naive gradient estimation by explicitly handling discontinuities in visibility, such as silhouette and shadow edges, which are critical for materials like BSSRDFs where opacity changes are blurred by subsurface scattering. Unlike [Reparameterized Volume Sampling](3309-reparameterized-volume-sampling.md) which focuses on volume density reparameterization, this method operates on surface path integrals with subsurface scattering.

### [Reparameterization](11557-reparameterization.md)
Reparameterization is a technique in differentiable volume rendering and path tracing that counteracts parameter-dependent silhouette motion and the resulting bias in differentiated rendering. It solves the problem where scene geometry parameters (e.g., vertex positions) influence the domain of integration or the support of the sampling distribution, causing discontinuities in the gradient estimation. By reparameterizing the ray directions, it freezes silhouette motion within spherical integrals, allowing gradients to flow through geometric intersections without bias. It sets itself apart from sibling methods by explicitly warping the ray space to handle moving boundaries, whereas methods like [Reparameterized Volume Sampling](3309-reparameterized-volume-sampling.md) focus on density field inversion.

### PGRIS Estimator
#### definition
- [S1] A gradient estimator that uses positivization to handle both positive and negative integrands, achieving theoretical zero-variance convergence of resampled derivative estimates.
#### comparison
- [S1] PGRIS uses the integrand as is (with positivization), whereas GRIS without positivization uses the absolute value of the integrand as the target function, leading to higher variance.

### Bidirectional Estimator for Implicit Surfaces
#### definition
- [S2] A Monte Carlo estimator for differentiable rendering of implicit surfaces that performs better than unidirectional estimators in scenes with hard-to-sample lights and specular materials.
#### pitfall
- [S2] Unidirectional estimators struggle with hard-to-sample lights and specular materials, leading to noisier gradients and less accurate reconstructions.
#### comparison
- [S2] Performs better than unidirectional estimators in scenes with hard-to-sample lights and specular materials, producing cleaner derivative images and smoother optimized shapes.

### Combined Estimator
#### formula
- [S3] The combined estimator is derived by plugging the Monte Carlo-based estimator (Equation 5) into the volume rendering estimator (Equation 3). Equation 5 is: $\hat {L}_\mathrm {o}(\xposof {t}, \dirout ) = \frac {1}{\mcount } \sum _{j=1}^{\mcount } f\lft (\xposof {t}, \superj {\dirin }, \dirout \rgt ) L_\mathrm {i}\lft (\xposof {t}, \superj {\dirin }\rgt ) \lft (\mathbf {n} \cdot \superj {\dirin } \rgt ) \lft / p\lft (\superj {\dirin }\rgt ) \rgt .$
#### parameter
- [S3] mcount: The number of samples used in the Monte Carlo summation for the estimator.
#### pitfall
- [S3] Equation 6 requires recursive evaluation of both the volume rendering integral and the rendering equation, which can lead to a rapid increase in computational cost if not handled carefully.

## Background
The reparameterization trick originated in probabilistic graphical models and variational autoencoders, where it was used to enable gradient-based optimization of latent variable models [G]. In computer graphics, it was adapted to differentiable rendering to address the non-differentiability of sampling distributions that depend on scene parameters. Early work focused on volume rendering, where the density field defines the sampling distribution [S1]. Subsequent developments extended the concept to path tracing, handling visibility discontinuities via boundary path integrals [S2] and silhouette motion via ray warping [S1, S3]. Recent advancements include the integration of ReSTIR (Resampled Importance Sampling) into differentiable rendering to reduce variance across optimization iterations [S1], and the application of these techniques to implicit surfaces and complex materials [S2, S3].

## Sources
[S1] Parameter-space ReSTIR for Differentiable and Inverse Rendering, https://doi.org/10.1145/3588432.3591512, fetched 2026-10-27
[S2] Path-Space Differentiable Rendering of Implicit Surfaces, https://doi.org/10.1145/3641519.3657473, fetched 2026-10-30
[S3] Flash Cache: Reducing Bias in Radiance Cache Based Inverse Rendering, https://doi.org/10.1007/978-3-031-73390-1_2, fetched 2026-10-27
