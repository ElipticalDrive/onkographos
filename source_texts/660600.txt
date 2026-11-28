## Overview
CPU path tracer developed in C++ on top of the Nori educational rendering framework, implementing and studying Monte Carlo techniques for physically based rendering and global illumination.
The project was developed incrementally, from sampling and direct lighting to a full path tracer with next event estimation, multiple importance sampling and microfacet materials.

## Disclaimer
The source code is not included in this repository because the project was developed on top of the Nori framework and is subject to its licensing and distribution restrictions.
This repository therefore contains the project documentation, rendered results and visual comparisons of the implemented techniques, while the implementation itself remains private.

## Sampling
* Tent distribution sampling
* Uniform hemisphere sampling
* Cosine-weighted sampling using both Malley's method and inverse transform sampling
* Statistical validation of sampling distributions using null hypothesis tests
* Area-weighted sampling of mesh triangles
<p align = "center">
  <img width="650" height="300" alt="cosinesampling" src="https://github.com/user-attachments/assets/00e44e08-d10a-40b7-81fc-63314bed9328" />
</p>

## Direct Lighting
* Material sampling
* Light sampling with visibility testing through shadow rays
* Conversion from area-based PDFs to solid-angle PDFs
* Luminance and area-weighted emitter selection to reduce estimator variance
<p align = "center">
  <img width="673" height="300" alt="matvslightsampling" src="https://github.com/user-attachments/assets/30f328a7-446c-49ad-b8bb-08723f9e50b1" />
</p>

## Path Tracing
* Recursive path tracer for global illumination
* Indirect illumination through multiple light bounces
* Next Event Estimation (NEE) for direct lighting at every bounce
* Fixed and throughput-based adaptive Russian roulette
* Support for perfect mirror materials
<p align = "center">
  <img width="678" height="300" alt="path" src="https://github.com/user-attachments/assets/609b491c-8e9c-43ae-99c7-d21a21f9b260" />
</p>

## Multiple Importance Sampling
* Multiple Importance Sampling (MIS) combining material and light sampling
* Balance, power and maximum heuristics
* Comparison of sampling strategies and their effect on estimator variance
* Improved convergence for scenes containing both diffuse and specular materials
<p align = "center">
  <img width="683" height="300" alt="MIS" src="https://github.com/user-attachments/assets/43abe61a-08d7-4e49-b09e-f7e9a16c4378" />
</p>

## Microfacet BRDF
* Cook–Torrance microfacet BRDF
* Combined diffuse and specular response
* Importance sampling of diffuse and specular lobes
<p align = "center">
  <img width="347" height="300" alt="BRDF" src="https://github.com/user-attachments/assets/4351c582-ad22-44b1-abf3-8c0c6b18dc93" />
</p>

## Additional Techniques
**Ambient Occlusion**
* Explicit ambient occlusion through visibility queries
* Comparison between manually applied AO and the occlusion effects naturally produced by global illumination
<p align = "center">
  <img width="680" height="300" alt="ao" src="https://github.com/user-attachments/assets/e8d1355b-809b-43ca-90dd-e002a1599723" />
</p>

**Tone Mapping**
* ACES-inspired tone mapping
* Luminance-based processing
* Gamma correction
<p align = "center">
  <img width="677" height="300" alt="tonemapping" src="https://github.com/user-attachments/assets/f512c01d-6a99-4b51-b1af-3943534a4d61" />
</p>

## Experiments
The project includes experiments comparing:
* Convergence at different spp counts
* Material sampling vs. light sampling
* Uniform vs. luminance and area-weighted emitter sampling
* Different MIS heuristics
* Convergence times with different Russian roulette approaches
* Direct lighting vs. direct + indirect illumination
* Different bounce limits
* Explicit vs. naturally occurring ambient occlusion
* NEE with and without MIS
* Effects of MIS on a Microfacet-Based BRDF

## Technologies
* C++
* Nori
