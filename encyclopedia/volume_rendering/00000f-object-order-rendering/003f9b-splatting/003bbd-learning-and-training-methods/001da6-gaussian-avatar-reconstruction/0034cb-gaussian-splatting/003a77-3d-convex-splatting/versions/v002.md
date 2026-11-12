# 3D Convex Splatting


## Overview
3D Convex Splatting is a neural rendering method that replaces standard 3D Gaussian primitives with smooth convex shapes to represent scenes [S1, unofficial]. It solves the limitation of Gaussians in accurately representing flat surfaces and sharp edges, offering greater flexibility and requiring fewer primitives for equivalent fidelity [S1, unofficial]. Unlike [Gaussian Splatting](946-gaussian-splatting.md), which relies on elliptical supports, this method introduces non-elliptical convex boundaries to better capture geometric details.

## How it works
**Inputs and Outputs:**
*   **Input:** A scene representation defined by a set of smooth convex primitives, each characterized by position, orientation, scale, and opacity/color attributes.
*   **Output:** A rendered 2D image.

**Data Structures:**
The specific internal data layout for the convex primitives is not given in the sources. The method relies on a representation that supports smooth convex boundaries, distinct from the covariance-based representation of Gaussians.

**Steps:**
1.  **Primitive Definition:** Define scene geometry using smooth convex shapes instead of 3D Gaussians. These shapes are designed to offer greater flexibility in modeling flat surfaces and sharp edges [S1, unofficial].
2.  **Rasterization:** Project the 3D convex primitives onto the 2D image plane. This step requires an efficient rasterizer capable of handling non-elliptical shapes.
3.  **Rendering:** Composite the projected primitives to produce the final image. The method is coupled with an efficient CUDA rasterizer to improve fidelity while reducing memory usage [S1, unofficial].

**Parameters:**
Typical values for parameters such as the number of primitives or specific convex shape constraints are not given in the sources.

**Edge Cases and Pitfalls:**
*   **Precision:** The sources do not specify precision issues or numerical stability concerns specific to convex splatting.
*   **Boundaries:** The method aims to better handle sharp edges, which are typically challenging for Gaussian splatting, but specific boundary handling algorithms are not detailed in the provided text.

## Subtopics

### [Compositional Splatting Strategy](13273-compositional-splatting-strategy.md)
A rendering strategy for non-elliptical primitives (Gaussian lines and triangles) that decomposes primitives into vertices and re-sketches the shape. It is chosen when mixed-primitive frameworks are needed for sharp edges.

## Operating characteristics
*   **Time and Memory Complexity:** The method reduces memory usage compared to standard Gaussian Splatting due to the efficiency of the convex representation and the coupled CUDA rasterizer [S1, unofficial]. Specific complexity formulas are not given.
*   **Measured Performance:** The method improves fidelity while reducing memory usage [S1, unofficial]. No specific benchmark numbers (FPS, memory MB) are provided in the sources.
*   **Strengths:** Greater flexibility than Gaussians; more accurate representation of flat surfaces and sharp edges; requires fewer primitives; reduced memory usage.
*   **Limitations:** The specific computational cost of rasterizing convex shapes compared to Gaussians is not quantified, though the use of a specialized CUDA rasterizer implies optimization for this task.
*   **When to use:** When accurate representation of flat surfaces and sharp edges is critical and fewer primitives are desired.
*   **When not to use:** When compatibility with standard Gaussian-based pipelines is required without modification.

## Background
3D Convex Splatting was introduced as the first work to move beyond Gaussian primitives by introducing smooth convex shapes [S1, unofficial]. It is part of a broader effort to bridge neural rendering with classical computer graphics, progressing from smooth convexes to triangles and finally meshes [S1, unofficial]. The work aims to integrate modern differentiable frameworks with decades of GPU-accelerated graphics techniques.

## Sources
[S1] Bridging neural rendering and classical computer graphics, https://core.ac.uk/works/363431310, fetched 2026-10-29 (unofficial)
