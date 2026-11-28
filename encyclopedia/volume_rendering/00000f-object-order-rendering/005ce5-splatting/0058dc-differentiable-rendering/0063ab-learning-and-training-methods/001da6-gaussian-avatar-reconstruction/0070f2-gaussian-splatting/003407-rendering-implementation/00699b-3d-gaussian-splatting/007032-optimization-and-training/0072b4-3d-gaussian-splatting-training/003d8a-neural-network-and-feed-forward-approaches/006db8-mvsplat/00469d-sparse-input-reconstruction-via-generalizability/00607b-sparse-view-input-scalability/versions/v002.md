# Sparse View Input Scalability

Also called: Arbitrary View Count Scalability, Progressive Multi-View Reconstruction

## Overview
Sparse View Input Scalability is the capability of generalizable neural rendering methods to process an arbitrary number of input views, with reconstruction quality progressively improving as more views are added [S1]. It leverages cost volume representations as input to attribute prediction networks for Gaussians, enabling direct inference from sparse calibrated inputs without per-scene optimization [S2, S2, unofficial].

## How it works
**Inputs and Outputs:**
- **Inputs:** A set of $N$ input images [S1].
- **Outputs:** A set of 3D Gaussian attributes predicted by a neural network, which are rendered via differentiable rasterization [G].

**Data Structures:**
- **Cost Volume:** A representation brought into sparse view reconstruction to serve as input for the attribute prediction network [S2, S2, unofficial].

**Steps:**
1. Construct a cost volume representation from the input views [S2, S2, unofficial].
2. Pass the cost volume to an attribute prediction network for Gaussians [S2, S2, unofficial].
3. The network outputs Gaussian attributes, which are rasterized to produce the output image [G].

**Formulas:**
- The Gaussian attributes $A$ are predicted by the network $f_\theta$ from the cost volume $C$:
  $$ A = f_\theta(C) $$
  where $C$ is the cost volume representation and $A$ denotes the predicted Gaussian attributes [G].

**Parameters:**
- **Number of Views ($N$):** The method supports an arbitrary number of input views, with experiments conducted using $N \in [2, 8]$ [S1].
- **Depth Planes:** The number of depth hypotheses used in cost volume construction. Not given in the sources.

**Edge Cases and Pitfalls:**
- **Few Views ($N=2$):** The method produces plausible geometries using only two views, but accuracy is lower than with more views [S1].
- **Degenerate Input:** If input views have insufficient overlap or extreme baselines, cost volume construction may fail to find consistent matches. Not given in the sources.
- **Precision:** Cost volume construction requires precise camera calibration. Not given in the sources.

## Operating characteristics
- **Time Complexity:** Inference time is constant with respect to scene complexity but depends on the number of views $N$ and the resolution of the cost volume. Not given in the sources.
- **Memory Complexity:** Memory usage scales with the size of the cost volume, which depends on the number of views, image resolution, and depth planes. Not given in the sources.
- **Performance:** On the DTU dataset (Scan105), the method produces plausible geometries with 2 views. Quality progressively improves and converges to a fairly low reconstruction error as views increase from 2 to 8 [S1].
- **Strengths:** Generalizable to unseen scenes; handles arbitrary view counts; produces smoother surfaces compared to density-based methods like MVSNerf [S1].
- **Limitations:** Performance degrades with very sparse views compared to dense inputs [S1].
- **When to Use:** Not given in the sources.
- **When Not to Use:** Not given in the sources.

## Background
The capability to handle arbitrary view counts with progressive quality improvement was demonstrated in SparseNeuS, which showed results from 2 to 8 views on the DTU dataset [S1]. The specific integration of cost volume representations into Gaussian Splatting for sparse view reconstruction was introduced by MVSplat, building on [Sparse Input Reconstruction via Generalizability](11788-sparse-input-reconstruction-via-generalizability.md) and earlier generalizable reconstruction work [S2, S2, unofficial].

## Sources
[S1] SparseNeuS: Fast Generalizable Neural Surface Reconstruction from Sparse Views, https://doi.org/10.1007/978-3-031-19824-3_13, fetched 2026-10-26
[S2] PDF Recent advances in 3D Gaussian splatting - Springer, https://link.springer.com/content/pdf/10.1007/s41095-024-0436-y.pdf, fetched 2026-10-29 (unofficial)
