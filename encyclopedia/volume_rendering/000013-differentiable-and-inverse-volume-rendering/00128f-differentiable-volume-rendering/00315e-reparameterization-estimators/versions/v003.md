# Reparameterization Estimators

Also called: Reparameterization Trick, Change of Variables for Gradients

## Overview
Reparameterization Estimators compute gradients of rendering integrals by transforming random variables such that their distribution is independent of the scene parameters $\theta$, allowing the derivative to pass through the sampling operation [S1, S2, S3]. This family solves the problem of high-variance or biased gradients caused by parameter-dependent support in Monte Carlo integration, such as moving silhouettes or density-dependent sampling locations [S1, S2, S3]. Unlike [Automatic Differentiation (Reverse/Adjoint Mode)](7151-automatic-differentiation-reverse-adjoint-mode.md), which differentiates the computational graph directly and suffers from discontinuities when the support changes, reparameterization estimators ensure the sampling distribution remains static, yielding lower-variance gradients [S1, S2, S3].

## Shared principles
The core mechanism relies on the change of variables formula for integrals. Let $I(\theta) = \int f(x, \theta) dx$ be the rendering integral where $x$ is a random variable with distribution $p(x|\theta)$. Direct differentiation $\frac{\partial}{\partial \theta} \int f(x, \theta) p(x|\theta) dx$ is problematic if the support of $p$ depends on $\theta$.

The reparameterization trick introduces a parameter-independent random variable $u \sim p(u)$ and a deterministic function $x = g(u, \theta)$ such that $x \sim p(x|\theta)$ [S1, S2, S3]. The integral becomes:
$$
I(\theta) = \int f(g(u, \theta), \theta) p(u) du
$$
The gradient is then:
$$
\nabla_\theta I(\theta) = \int \nabla_\theta \left[ f(g(u, \theta), \theta) \right] p(u) du
$$
This allows the gradient to be estimated by sampling $u$ independently of $\theta$ and differentiating the transformed integrand [S1, S2, S3]. Key requirements include:
1.  **Differentiability:** The mapping $g(u, \theta)$ and the integrand $f$ must be differentiable with respect to $\theta$ [S1, S2, S3].
2.  **Support Independence:** The domain of $u$ must not depend on $\theta$ [S1, S2, S3].
3.  **Jacobian Handling:** If the transformation involves a change of measure, the Jacobian determinant must be included in the integrand [S1, S2, S3].

## Subtopics

### [Reparameterized Volume Sampling](3309-reparameterized-volume-sampling.md)
Reparameterized Volume Sampling computes color estimates by sampling points on a ray using the inverse of the cumulative density function (CDF), allowing gradients to back-propagate through the sampling locations with respect to the density field. It sets itself apart from NeRF's hierarchical sampling by learning sampling points end-to-end without auxiliary coarse models, and exhibits lower variance than standard importance sampling with uniform distributions.

### [Boundary Path Integral Formulation](3389-boundary-path-integral-formulation.md)
The Boundary Path Integral Formulation is a method for estimating the derivatives of the path integral in differentiable volume rendering with respect to object shape parameters. It solves the problem of computing gradients when scene geometry depends on parameters $\theta$ by separating the differentiation of the path integral into interior and boundary terms. Unlike its siblings [Reparameterized Volume Sampling](3309-reparameterized-volume-sampling.md) and [Reparameterized Monte Carlo Volume Rendering](4798-reparameterized-monte-carlo-volume-rendering.md), which typically rely on differentiable sampling within the volume or standard CDF inversion, this method converts interior volume integrals into boundary area integrals using the divergence theorem. This eliminates the need to evaluate the deformation mapping inside the volume, avoiding expensive tetrahedralizations.

### [Reparameterized Monte Carlo Volume Rendering](4798-reparameterized-monte-carlo-volume-rendering.md)
Reparameterized Monte Carlo Volume Rendering is a differentiable rendering method that estimates expected radiance by sampling from the density-induced probability distribution using the reparameterization trick, allowing end-to-end gradient-based optimization without auxiliary models. It sets itself apart from standard Monte Carlo volume rendering by decoupling the randomness from the scene parameters via the inverse cumulative distribution function (CDF) of the opacity, ensuring that gradients flow directly through the sampling process. Compared to its sibling [Reparameterized Volume Sampling](3309-reparameterized-volume-sampling.md), which samples uniformly in volume space, R/MC samples in opacity space, concentrating samples in denser regions of the scene.

### [Differential path integral estimator with MIS and NEE](6332-differential-path-integral-estimator-with-mis-and-nee.md)
This method computes the gradient of the rendered image with respect to scene parameters $\theta$ by integrating differential measurement contributions over path space. It solves the problem of high variance in gradient estimation for specular materials and complex lighting by combining standard BRDF sampling with Next Event Estimation (NEE) using Multiple Importance Sampling (MIS). Unlike standard NEE which only connects paths to emitters at the final vertex, this method allows the differential vertex (the location where the parameter derivative is applied) to be located at any intermediate vertex $i$ or at the emitter connection, combining BRDF and emitter sampling probabilities into a single mixture weight. It builds on [Reparameterization Estimators](6499-reparameterization-estimators.md) by extending the reparameterization principle to full path-space integrals with explicit MIS weights for differential contributions.

### [Reparametrized Integral Gradients](6785-reparametrized-integral-gradients.md)
Reparametrized Integral Gradients is a technique for differentiating rendering integrals with discontinuous integrands by applying changes of variables that remove the dependence of discontinuity locations on scene parameters. Unlike methods that explicitly sample edges or decompose integrals into interior and boundary terms, this approach warps the integration domain so that moving discontinuities become stationary in the transformed space, allowing differentiation under the integral sign without explicit boundary sampling. It sets itself apart from [Reparameterized Volume Sampling](3309-reparameterized-volume-sampling.md) by focusing on general visibility discontinuities (edges) rather than just volume density variations, and from [Boundary Path Integral Formulation](3389-boundary-path-integral-formulation.md) by avoiding the explicit estimation of boundary terms.

### [Plateau-Reduced Differentiable Path Tracing](8826-plateau-reduced-differentiable-path-tracing.md)
Plateau-Reduced Differentiable Path Tracing (PR-DPT) is an inverse rendering technique that alleviates optimization plateaus (regions of zero gradient) by convolving the rendering equation with a smoothing kernel over the parameter space. Unlike standard differentiable path tracers that suffer from discontinuous gradients due to visibility changes, PR-DPT creates a smooth objective function allowing gradient-based optimizers to converge without requiring the renderer's internal gradient tape or special integrators for discontinuities. It builds on [Reparameterization Estimators](6499-reparameterization-estimators.md) by applying a convolutional blur to the parameter space rather than just the path space.

### [Monte Carlo Estimation of Differentiable Path Integrals](10814-monte-carlo-estimation-of-differentiable-path-integrals.md)
This method estimates the gradient of a path integral rendering equation with respect to scene parameters $\pi$ by decomposing the derivative into interior contributions (derivative of the path contribution function) and boundary contributions (derivative of the visibility boundary). It distinguishes itself from sibling methods by explicitly handling discontinuities in translucent materials (BSSRDF) via secondary visibility boundaries, whereas opaque surface methods typically only project shadow edges directly to the camera.

### [Reparameterization](11557-reparameterization.md)
Reparameterization is a technique in differentiable rendering that counteracts parameter-dependent silhouette motion and the resulting bias in differentiated rendering integrals. It solves the problem where gradients with respect to scene geometry parameters are biased because the intersection points (silhouettes) move discontinuously as parameters change. It sets itself apart from sibling methods by explicitly constructing a warp field via auxiliary rays to "freeze" discontinuities within spherical integrals, reducing to the identity operation during primal rendering while expanding into complex gradient-scattering operations during differentiation.

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
Reparameterization estimators originate from the broader field of stochastic gradient estimation, particularly in variational inference [G]. In computer graphics, they were adapted to differentiable rendering to handle the discontinuities inherent in ray tracing, such as visibility changes and specular reflections. The specific application to volume rendering and path tracing has evolved to address the high variance of naive AD approaches [S1, S2, S3]. Key developments include the use of CDF inversion for volume sampling [S1], the divergence theorem for boundary terms in shape optimization [S2], and the warping of integration domains for visibility discontinuities [S1, S3]. These methods are widely used in inverse rendering, neural radiance fields (NeRFs), and differentiable geometry optimization [S1, S2, S3].

## Sources
[S1] Parameter-space ReSTIR for Differentiable and Inverse Rendering, https://doi.org/10.1145/3588432.3591512, fetched 2026-10-27
[S2] Path-Space Differentiable Rendering of Implicit Surfaces, https://doi.org/10.1145/3641519.3657473, fetched 2026-10-30
[S3] Flash Cache: Reducing Bias in Radiance Cache Based Inverse Rendering, https://doi.org/10.1007/978-3-031-73390-1_2, fetched 2026-10-27
