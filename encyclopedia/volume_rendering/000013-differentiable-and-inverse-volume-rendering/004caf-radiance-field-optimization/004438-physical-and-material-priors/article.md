# Physical and Material Priors


## Overview
Physical and Material Priors methods incorporate physical laws (such as ray refraction) or material properties into the differentiable rendering pipeline to recover underlying scene properties like geometry, lighting, and optical constants [G]. They are distinguished from siblings by their focus on recovering physical optical properties rather than just appearance, building on [Radiance-field optimization](44-radiance-field-optimization.md).

## Shared principles

## Subtopics

### [Refractive Index Tomography using Neural Fields](4146-refractive-index-tomography-using-neural-fields.md)
Recovers a 3D refractive index field and luminance from single-view refracted images by optimizing neural fields. It differs from standard methods by modeling light propagation through varying refractive indices using ODEs instead of fixed-step volumetric integration. Chosen when recovering optical constants from refracted measurements is required.

### [Inverse Rendering and Optical Properties](7914-inverse-rendering-and-optical-properties.md)
Reconstructs geometry, material properties, and lighting by optimizing differentiable rendering parameters. It is distinguished by its focus on recovering physical optical properties (e.g., extinction coefficients, reflectance) rather than just static radiance. Chosen when optimizing underlying optical properties is required.

### [Neural Radiative Transfer via Physics-Informed Optimization](10842-neural-radiative-transfer-via-physics-informed-optimization.md)
Reconstructs spatially varying optical properties (absorption, scattering, phase function) and the light field of participating media from posed RGB images by enforcing the Radiative Transfer Equation (RTE) as a physics-informed constraint. It differs from surface-based inverse rendering by enforcing light transport locally throughout the volume. Chosen when physically complete volumetric light transport is required without restrictive assumptions such as homogeneous media.

### [Multi-Sensor and Physical Priors](11031-multi-sensor-and-physical-priors.md)
Extends differentiable volume rendering by integrating physical sensor models and geometric priors to solve inverse problems where optical assumptions fail or data is sparse. It differs from standard radiance field optimization by replacing or augmenting the emission-absorption model with physics-based signal formation equations. Chosen when optical assumptions fail or data is sparse.

### Joint Optimization of Triangle Mesh, Material, and Light (JOC)
A framework that jointly optimizes triangle mesh, Physically-Based (PBR) material, and High Dynamic Range (HDR) light probe from pre-trained neural fields by combining neural field reconstruction with differentiable rasterization/ray tracing [S1, unofficial]. It obtains a Neural Radiance Cache (NRC) from a pre-trained neural field to model indirect illumination, uses the trained neural field to initialize Differentiable Marching Tetrahedrons (DMTet), and integrates the NRC into a ray tracing pipeline to compute global illumination via single-bounce differentiable MC ray tracing [S1, unofficial]. Unlike previous inverse rendering methods based on textured meshes that bake indirect illumination into materials, JOC separates indirect illumination effects from material [S1, unofficial]. Unlike NVDIFFRECMC which only considers direct lighting due to noise and iteration time costs of multi-bounce path tracing, JOC captures global illumination [S1, unofficial]. The input neural field must be capable of synthesizing novel views and estimating depth via ray casting [S1, unofficial].

## Background
Not given in the sources.

## Sources
[S1] Joint Optimization of Triangle Mesh, Material, and Light from Neural Fields with Neural Radiance Cache, https://doi.org/10.48550/arxiv.2305.16800, fetched 2026-10-26 (unofficial)
