# Volumetric Path Tracing with Participating Media

## Overview

This project explores **volumetric path tracing** as a physically based rendering technique for simulating participating media such as fog, smoke, and clouds. Unlike classical path tracing, which assumes a vacuum medium, this implementation models light transport *inside volumes*, enabling accurate simulation of scattering phenomena.

The project was developed in an academic context to study how volumetric path tracing compares to ray marching approaches in terms of realism, complexity, and performance.

## Motivation

Participating media introduce additional realism to rendered scenes but significantly increase complexity. Modeling how light scatters within a volume requires extending traditional surface-based path tracing to support random walks inside the medium. This project investigates that extension and its implications.

## Key Concepts

* Participating media (fog, smoke, clouds)
* Volumetric path tracing
* Multiple scattering
* Physically based light transport
* Variance vs. realism trade-offs

## Methodology

The renderer extends classical path tracing by allowing rays to:

* Enter volumetric regions
* Sample scattering events within the medium
* Continue random walks after each scattering interaction

This approach naturally supports **multiple scattering**, leading to more physically accurate volumetric effects compared to single-scattering or ray marching techniques.

## Comparison with Ray Marching

| Aspect              | Ray Marching | Volumetric Path Tracing |
| ------------------- | ------------ | ----------------------- |
| Accuracy            | Approximate  | Physically based        |
| Multiple Scattering | Limited      | Fully supported         |
| Variance            | Low          | Higher                  |
| Performance         | Faster       | More expensive          |



## References

[1] Physically Based Rendering: From Theory to Implementation

---

Feel free to explore the code and the results ( https://drive.google.com/drive/folders/1R71_UK8Q61WIc06NvWc8uHgEusKIBMql?usp=drive_link) .
