# Coarse Grid for Large Models
Also called: Lambda 1 for Large Models

## Overview
This method selects a significantly coarser voxel grid resolution (lower $\lambda$) for very large triangle models to minimize grid rebuild time, accepting increased ray traversal overhead to achieve better aggregate frame rates. It differs from the standard [Grid Resolution Selection via Lambda](3270-grid-resolution-selection-via-lambda.md) approach, which targets moderate scenes with $\lambda \approx 5$, by prioritizing build speed over traversal efficiency for models exceeding several million triangles.

## How it works
The method modifies the grid resolution parameter $\lambda$ based on the triangle count of the scene. It does not alter the grid construction algorithm itself but changes the input parameter to the [Grid Resolution Selection via Lambda](3270-grid-resolution-selection-via-lambda.md) method.

**Inputs:**
- Triangle count of the model ($N_{tri}$).
- Scene bounding box (used by the parent lambda selection method).

**Outputs:**
- Grid resolution parameters ($\lambda_x, \lambda_y, \lambda_z$) passed to the grid builder.

**Logic:**
1. Determine if the model is "significantly large" (typically $> 1$ million triangles).
2. If the model is moderate-sized, use the default $\lambda \approx 5$ as determined by [Grid Resolution Selection via Lambda](3270-grid-resolution-selection-via-lambda.md).
3. If the model is very large, set $\lambda = 1$ for all dimensions.
4. Construct the grid using the selected $\lambda$.

**Parameters:**
- $\lambda = 1$: The coarse resolution factor for large models.
- $\lambda = 5$: The default resolution factor for moderate models [S1, S2].

**Edge Cases:**
- For models between moderate and very large sizes, the trade-off between rebuild time and render time must be evaluated. The sources indicate that for the Thai Statue (10M triangles), $\lambda=5$ rebuild time was 3x the ray tracing time, making $\lambda=1$ optimal [S1, S2].
- The method assumes single-threaded rebuild for the reported performance metrics, though multi-threading can further reduce rebuild time [S1, S2].

## Operating characteristics
- **Time Complexity:** Grid rebuild time decreases significantly with lower $\lambda$. For the 10M triangle Thai Statue, $\lambda=1$ rebuild is much faster than $\lambda=5$, resulting in total frame times (including rebuild) of less than 1.5 seconds per frame [S1, S2].
- **Memory Complexity:** Coarser grids use less memory, though specific memory savings are not quantified in the sources.
- **Performance:** For the 2.2M triangle Soda Hall model, $\lambda=1$ achieves 3.5 frames/second including rebuild (110ms rebuild time) [S1, S2]. This maintains interactivity for large models where finer grids would cause excessive rebuild latency.
- **Strengths:** Enables interactive frame rates for extremely large models by balancing build and render costs.
- **Limitations:** Increased ray traversal time due to lower grid resolution. Not suitable for moderate-sized scenes where $\lambda \approx 5$ is optimal [S1, S2].

## Background
This technique is described in the paper "Ray Tracing Animated Scenes using Coherent Grid Traversal" by Wald et al. (2005/2006) [S1, S2]. It addresses the scalability issues of grid-based ray tracing when applied to models with millions of triangles, where the cost of building a fine grid outweighs the benefits of faster ray traversal.

## Sources
[S1] Ray Tracing Animated Scenes using Coherent Grid Traversal, https://www.sci.utah.edu/~thiago/papers/grid.pdf, fetched 2026-10-25
[S2] Ray Tracing Animated Scenes using Coherent Grid Traversal, https://www.sci.utah.edu/~wald/Publications/2006/CGT/grid.pdf, fetched 2026-10-25
