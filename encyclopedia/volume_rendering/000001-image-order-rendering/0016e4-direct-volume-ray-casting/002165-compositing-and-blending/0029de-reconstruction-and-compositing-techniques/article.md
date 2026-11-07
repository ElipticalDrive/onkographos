# Reconstruction and Compositing Techniques


## Overview
This family encompasses methods that bridge the gap between discrete volumetric data and the continuous optical model required for ray casting, specifically focusing on how input samples are reconstructed into a continuous field and how those fields are composited along rays. It distinguishes itself from sibling methods by addressing the mathematical formulation of the volume function evaluation (algebraic vs. spatial) and the weighting of reconstruction accuracy during opacity accumulation.

## Shared principles
The family operates on the premise that raw discrete volume data $V(i,j,k)$ must be mapped to a continuous function $f(x,y,z)$ to support arbitrary ray parameterizations and accurate integration. This mapping is achieved through a reconstruction kernel or polynomial basis, which is then integrated along the ray path. The resulting continuous values are used to compute color and opacity contributions, which are subsequently accumulated using the standard compositing equations defined in [Compositing and Blending](2515-compositing-and-blending.md).

## Subtopics

### [Composition via Convolution (Power Form)](1591-composition-via-convolution-power-form.md)
A method that evaluates the composition of a trivariate polynomial field with a linear ray parameterization, transforming a 3D function into a univariate function along the ray. It solves the problem of efficient numerical evaluation in direct volume ray casting by using convolution operators to reduce computation and register usage compared to naive substitution. It is distinguished from its sibling [Feed-Backward Reconstruction](2433-feed-backward-reconstruction.md) by operating in the algebraic domain using polynomial composition rather than spatial-domain discrete convolution.

### [Feed-Backward Reconstruction](2433-feed-backward-reconstruction.md)
A spatial-domain resampling method that computes each output sample by gathering weighted contributions from neighboring input samples. It solves the problem of mapping discrete 3D volume data to a continuous ray parameterization or output grid by centering a reconstruction kernel at the output location and integrating the input signal. It is distinguished from its sibling [Composition via Convolution (Power Form)](1591-composition-via-convolution-power-form.md) by operating in the spatial domain using discrete convolution kernels rather than composing trivariate polynomials in the algebraic domain.

### Source-Attenuation Reprojection
**Definition:** A technique by Vannier et al. that assigns a source strength and an attenuation coefficient (opacity) to each voxel [S1, unofficial].
**Steps:**
1. Trace rays through the volume array until they hit a surface [S1, unofficial].
2. Assign an intensity inversely proportional to the distance traveled along the ray [S1, unofficial].
3. Define a surface as a significant change in opacity value [S1, unofficial].
**Note:** This approach is threshold-based and suffers from associated artifacts [S1, unofficial].

### Reconstruction Validity Weighted Volume Rendering
**Definition:** A volume rendering technique that uses a reconstruction validity metric $\Phi(x, f, \Omega)$ to weight the opacity accumulation during ray casting, improving classification visualization by accounting for reconstruction accuracy in different density regions [S2].
**Steps:**
1. Calculate reconstruction validity $\Phi(x, f, \Omega)$ for each feature $f$ at each reconstruction point $x$ [S2].
2. For each ray, perform $K$ reconstructions (one for each feature class) to get source color $C_{src}$ and opacity $\alpha_{src}$ based on transfer functions [S2].
3. Perform back-to-front compositing along the ray, updating accumulated color $C_s$ and opacity $\alpha_s$ using the weighted opacity formula involving $\Phi(x, f, \Omega)$ [S2].
4. Repeat compositing for all $K$ features, as $\Phi$ is feature-dependent [S2].
**Formula:**
$$ C_s = C_s + (1 - \alpha_s) C_{src} $$
$$ \alpha_s = \alpha_s + (1 - \alpha_s) \alpha_{src} \Phi(x, f, \Omega) $$
where $C_s$ and $\alpha_s$ are accumulated color and opacity, $C_{src}$ and $\alpha_{src}$ are source color and opacity from the transfer function, and $\Phi$ is the reconstruction validity weight [S2].

## Background
The need for robust reconstruction and compositing techniques arises from the mismatch between discrete sensor data and continuous optical physics. Early medical visualization research, such as that by Vannier et al., explored source-attenuation reprojection to handle depth shading and partial obscuration [S1, unofficial]. More recent developments, such as the work by Zhang and Tang (2019), have integrated probabilistic classification models with kernel regression to define reconstruction validity, aiming to improve the visual separation of heterogeneous features in volume data [S2].

## Sources
[S1] Volumetric rendering of medical data : applications to stereotactic neurosurgery planning, https://doi.org/10.82308/10070, fetched 2026-10-26 (unofficial)
[S2] Volume Data Classification Visualization Based on Probabilistic Classification Model, https://doi.org/10.12677/csa.2019.911223, fetched 2026-10-30
