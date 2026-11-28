# Order-Dependent Sampling Strategies

Also called: Order-Dependent Sampling

## Overview
Order-Dependent Sampling Strategies are a category of volume illumination approaches designed to resolve the order-dependence of sampling operations required for the numerical solutions of integrals in volumetric lighting [S1, unofficial]. Unlike standard direct volume rendering which assumes a fixed view-dependent order, these methods address scenarios where illumination calculations depend on the specific sequence of samples relative to light sources [S1, unofficial]. They are distinguished from siblings like [Monte Carlo Volume Rendering](2060-monte-carlo-volume-rendering.md) by their deterministic handling of sample ordering for illumination integrals, and from [Ray Traversal and Integration Schemes](8341-ray-traversal-and-integration-schemes.md) by their focus on pre-processing or iterative ordering rather than traversal optimization alone.

## Shared principles
All methods in this family operate on the principle that the illumination integral (Equation 2 in [S1, unofficial]) is order-dependent with respect to the sampling operation [S1, unofficial]. The core mechanism involves resolving this dependency through one of two primary strategies:

1.  **Iterative Processing**: Determining a common sampling front based on camera and light positions to process all samples in the correct order, ensuring each sample is processed only once [S1, unofficial].
2.  **Precomputation**: Performing precomputation for a sufficient number of samples and storing results in data structures such as illumination volumes or spherical harmonic representations, making them available during rendering [S1, unofficial].

## Subtopics

### Iterative Processing
Iterative processing determines a common sampling front based on camera and light positions to process all samples in the correct order, whereby each sample is only processed once [S1, unofficial]. This approach is typically implemented in combination with slice-based volume rendering [S1, unofficial]. It is chosen when strict ordering is required for single-scattering or specific illumination models where precomputation is too memory-intensive. However, it prevents the use of early ray termination and empty space skipping, which are mandatory techniques for costly sampling operations [S1, unofficial].

### Precomputation
Precomputation facilitates solving the order-dependent illumination integral by performing calculations for a sufficient number of samples in advance, storing results in data structures ranging from simple illumination volumes to spherical harmonic representations [S1, unofficial]. This method is viable for single light source scenarios but becomes non-viable for multimodal data with multiple light sources [S1, unofficial]. Storing illumination information for all light sources and wavelengths results in huge data structures, and unnecessary computations are performed for occluded parts of the volumes since the precomputation is performed independently of the view-dependent visibility [S1, unofficial].

## Background
The concept of order-dependent sampling arises in the context of multimodal volume illumination, where standard volume rendering techniques fail to provide the desired interactive experience due to the complexity of illumination integrals [S1, unofficial]. The specific strategies of iterative processing and precomputation were identified and analyzed in the context of resolving order-dependence for advanced volumetric illumination [S1, unofficial]. The limitations of these approaches, particularly regarding memory usage for multiple light sources and the inability to use early ray termination, motivated the development of alternative approaches for multimodal volume illumination [S1, unofficial].

## Sources
[S1] Multimodal Volume Illumination, https://www.uni-ulm.de/fileadmin/website_uni_ulm/iui.inst.100/institut/Papers/viscom/2015/sunden15multimodal.pdf, fetched 2026-10-29 (unofficial)
