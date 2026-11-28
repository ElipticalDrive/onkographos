# Reparameterization Estimators


## Overview
Reparameterization estimators compute gradients of rendering integrals by transforming the integration domain or sampling distribution so that parameter dependence moves from domain boundaries or probability density functions into the integrand's argument [G]. This allows differentiation under the integral sign without boundary terms [G]. Unlike [Automatic Differentiation (Reverse/Adjoint Mode)](7151-automatic-differentiation-reverse-adjoint-mode.md) which propagates gradients through the computational graph, or [Analytical Gradients](7613-analytical-gradients.md) which derive closed-form derivatives, reparameterization relies on a change of variables to make discontinuities or sampling distributions static with respect to optimization variables [G].

## Shared principles
The core mechanism applies the change of variables formula to the rendering integral $I(\theta) = \int_{\Omega_\theta} f(x, \theta) dx$. By introducing a differentiable transformation $T(\omega, \theta): \Omega \to \Omega_\theta$ from a fixed reference domain $\Omega$ to the parameter-dependent domain $\Omega_\theta$, the integral becomes:
$$
I(\theta) = \int_{\Omega} f(T(\omega, \theta), \theta) \left| \det \frac{\partial T}{\partial \omega} \right| d\omega
$$
The gradient with respect to parameters $\theta$ is obtained by differentiating under the integral sign:
$$
\frac{\partial I}{\partial \theta} = \int_{\Omega} \frac{\partial}{\partial \theta} \left[ f(T(\omega, \theta), \theta) \left| \det \frac{\partial T}{\partial \omega} \right| \right] d\omega
$$
In Monte Carlo estimation, this is implemented by sampling points $\omega$ from a fixed distribution $p(\omega)$ on $\Omega$, mapping them to $x = T(\omega, \theta)$, and weighting by the Jacobian determinant $\left| \det \frac{\partial T}{\partial \omega} \right|$. The gradient estimator is:
$$
\hat{\nabla}_\theta I = \frac{1}{N} \sum_{i=1}^N \nabla_\theta \left[ \frac{f(T(\omega_i, \theta), \theta) \left| \det \frac{\partial T}{\partial \omega} \right|}{p(\omega_i)} \right]
$$
This requires computing the derivative of the transformation $T$ and the Jacobian determinant with respect to $\theta$ [G].

## Subtopics

### [Reparameterized Volume Sampling](3309-reparameterized-volume-sampling.md)
Computes color estimates by sampling points on a ray using the inverse of the cumulative density function (CDF), allowing gradients to back-propagate through sampling locations with respect to the density field. It differs from NeRF's hierarchical sampling by learning sampling points end-to-end without auxiliary networks, and exhibits lower variance than standard importance sampling with a uniform distribution. Choose when training neural radiance fields or density fields end-to-end.

### [Reparameterized Monte Carlo Volume Rendering](4798-reparameterized-monte-carlo-volume-rendering.md)
Estimates expected radiance by sampling from the density-induced probability distribution using the reparameterization trick, transforming the integration domain from ray distance to cumulative opacity. It sets itself apart by explicitly defining the sampling distribution via the ray's opacity function rather than general volume elements. Choose for unbiased gradient estimation through the inverse opacity function.

### [Differential path integral estimator with MIS and NEE](6332-differential-path-integral-estimator-with-mis-and-nee.md)
Computes the gradient of the rendered image with respect to scene parameters by integrating differential measurement contributions over path space, using Multiple Importance Sampling (MIS) to combine BRDF and emitter sampling, and Next Event Estimation (NEE) to connect paths to emitters. It solves high variance in scenes with varying material properties by allowing the differential vertex to be located at any intermediate vertex or emitter connection. Choose for scenes with complex material properties and varying light transport.

### [Reparametrized Integral Gradients](6785-reparametrized-integral-gradients.md)
Eliminates bias in gradients caused by visibility discontinuities moving with scene parameters by applying a change of variables to transform the integration domain such that discontinuities become static. It differs from explicit edge sampling methods by avoiding boundary terms, making it computationally efficient for complex scenes, though it approximates gradients and has limitations with specular materials. Choose for complex scenes where boundary detection is expensive.

### [Plateau-Reduced Differentiable Path Tracing](8826-plateau-reduced-differentiable-path-tracing.md)
Alleviates optimization plateaus by convolving the rendering equation with a smoothing kernel over the parameter space. Unlike standard differentiable path tracers, it ensures scene parameters continuously influence the objective function in expectation, allowing convergence on problems with intricate light transport like caustics. Choose when facing optimization plateaus or discontinuous gradients due to visibility changes.

### [Reparameterization](11557-reparameterization.md)
Counteracts parameter-dependent silhouette motion and resulting bias by applying a change of variables to freeze discontinuities in a reference domain. It sets itself apart by explicitly constructing a warp field via auxiliary ray tracing to handle the geometric transformation. Choose when handling moving geometry boundaries that cause non-differentiable jumps.

### [Control Variates for Gradient Variance Reduction](14988-control-variates-for-gradient-variance-reduction.md)
Reduces gradient estimator variance by leveraging the statistical property that the expected gradient of the weight term is zero. It sets itself apart by explicitly constructing a correlated control estimator from the Jacobian weights of the reparameterization. Choose when the integrand is smooth or constant and variance reduction is critical.

### PGRIS Estimator
- **Definition**: A gradient estimator that uses positivization to handle both positive and negative integrands, achieving theoretical zero-variance convergence of resampled derivative estimates [S1].
- **Comparison**: PGRIS uses the integrand as is (with positivization), whereas GRIS without positivization uses the absolute value of the integrand as the target function, leading to higher variance [S1].

### Bidirectional Estimator for Implicit Surfaces
- **Definition**: A Monte Carlo estimator for differentiable rendering of implicit surfaces that performs better than unidirectional estimators in scenes with hard-to-sample lights and specular materials [S2].
- **Pitfall**: Unidirectional estimators struggle with hard-to-sample lights and specular materials, leading to noisier gradients and less accurate reconstructions [S2].
- **Comparison**: Performs better than unidirectional estimators in scenes with hard-to-sample lights and specular materials, producing cleaner derivative images and smoother optimized shapes [S2].

### Combined Estimator
- **Formula**: The combined estimator is derived by plugging the Monte Carlo-based estimator (Equation 5) into the volume rendering estimator (Equation 3). Equation 5 is: $\hat {L}_\mathrm {o}(\xposof {t}, \dirout ) = \frac {1}{\mcount } \sum _{j=1}^{\mcount } f\lft (\xposof {t}, \superj {\dirin }, \dirout \rgt ) L_\mathrm {i}\lft (\xposof {t}, \superj {\dirin }\rgt ) \lft (\mathbf {n} \cdot \superj {\dirin }\rgt ) \lft / p\lft (\superj {\dirin }\rgt ) \rgt .$ [S3].
- **Parameter**: $mcount$: The number of samples used in the Monte Carlo summation for the estimator [S3].
- **Pitfall**: Equation 6 requires recursive evaluation of both the volume rendering integral and the rendering equation, which can lead to a rapid increase in computational cost if not handled carefully [S3].

## Background
Reparameterization estimators originate from the broader field of differentiable programming and stochastic gradient estimation, where the reparameterization trick is used to lower the variance of gradient estimates [G]. In computer graphics, the application to rendering was driven by the need to handle visibility discontinuities (silhouettes and shadows) which cause non-differentiable jumps in the rendering integral when scene parameters change [G]. The method is particularly relevant for inverse rendering tasks where geometry or material parameters must be optimized from images [G]. Recent developments include adapting ReSTIR to differentiable rendering for parameter-space reformulation [S1], applying path-space differentiable rendering to implicit surfaces [S2], and reducing bias in radiance cache-based inverse rendering [S3].

## Sources
[S1] Parameter-space ReSTIR for Differentiable and Inverse Rendering, https://doi.org/10.1145/3588432.3591512, fetched 2026-10-27
[S2] Path-Space Differentiable Rendering of Implicit Surfaces, https://doi.org/10.1145/3641519.3657473, fetched 2026-10-30
[S3] Flash Cache: Reducing Bias in Radiance Cache Based Inverse Rendering, https://doi.org/10.1007/978-3-031-73390-1_2, fetched 2026-10-27
