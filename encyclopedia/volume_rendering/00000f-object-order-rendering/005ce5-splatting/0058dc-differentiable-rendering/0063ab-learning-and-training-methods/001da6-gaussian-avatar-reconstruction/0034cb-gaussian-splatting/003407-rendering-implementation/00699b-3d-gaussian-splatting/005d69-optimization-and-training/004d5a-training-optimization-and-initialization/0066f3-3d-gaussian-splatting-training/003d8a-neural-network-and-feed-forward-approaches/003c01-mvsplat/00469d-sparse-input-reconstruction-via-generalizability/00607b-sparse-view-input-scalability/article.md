# Sparse View Input Scalability

Also called: Arbitrary View Count Scalability, MVSplat, Progressive Multi-View Reconstruction, Sparse Input Reconstruction via Generalizability

## Overview
Sparse View Input Scalability is the capability of a generalizable neural rendering method to handle an arbitrary number of input views, where reconstruction quality improves progressively as more views are added [S1]. It distinguishes itself from single-view methods (like PixelSplat or SplatterImage) by leveraging multi-view geometric constraints via cost volumes, and from iterative optimization methods by enabling direct inference from sparse inputs without per-scene tuning [S2, S2, unofficial].

## How it works
**Inputs and Outputs:**
- **Inputs:** A set of $N$ input images (where $N \ge 2$), typically calibrated with intrinsic and extrinsic parameters.
- **Outputs:** A set of 3D Gaussian attributes (mean, covariance, opacity, spherical harmonics coefficients) defined in a canonical space, which can be rendered via differentiable rasterization.

**Data Structures:**
- **Cost Volume:** A 3D tensor representing geometric consistency across views. For a target view, features from source views are warped into the target view's coordinate system. The volume aggregates these features to identify depth hypotheses.
- **Attribute Prediction Network:** A neural network (typically a U-Net or similar encoder-decoder) that takes the cost volume and/or image features as input and outputs Gaussian attributes.

**Steps:**
1. **Feature Extraction:** Extract multi-scale features from each input image using a shared encoder (e.g., ResNet or ConvNet) [S2, S2, unofficial].
2. **Cost Volume Construction:**
   - For a target view, warp features from source views into the target view's space using estimated depth hypotheses or inverse depth planes.
   - Aggregate the warped features and the target view's own features to form a cost volume. This volume encodes geometric consistency across the $N$ views [S2, S2, unofficial].
3. **Attribute Prediction:**
   - Pass the cost volume (and potentially the original image features) through the attribute prediction network.
   - The network outputs the parameters for each Gaussian: position ($\mu$), covariance ($\Sigma$), opacity ($\alpha$), and color/diffuse properties (SH coefficients) [S2, S2, unofficial].
4. **Rendering:**
   - Render the predicted Gaussians using standard differentiable splatting to produce the output image. The rendering loss is used during training of the network [S2, S2, unofficial].

**Formulas:**
- The cost volume $C$ at depth $d$ for a pixel $p$ in the target view is computed by aggregating features from source views $v_i$:
  $$ C(p, d) = \text{Aggregate}(F_{target}(p), F_{source}(warp(p, d, v_i))) $$
  where $warp$ projects the point from the target view to the source view using camera geometry and depth $d$ [S2, S2, unofficial].
- The Gaussian attributes $A$ are predicted by the network $f_\theta$:
  $$ A = f_\theta(C, F_{images}) $$
  where $A$ includes $\mu, \Sigma, \alpha, \text{SH}$ [S2, S2, unofficial].

**Parameters:**
- **Number of Views ($N$):** The method supports $N \ge 2$. Typical experiments use $N \in [2, 8]$ [S1].
- **Depth Planes:** The number of depth hypotheses used in cost volume construction. Not given in the sources.

**Edge Cases and Pitfalls:**
- **Few Views ($N=2$):** The method can produce plausible geometries with only 2 views, but accuracy is lower than with more views [S1].
- **Degenerate Input:** If input views have insufficient overlap or extreme baselines, cost volume construction may fail to find consistent matches. Not given in the sources.
- **Precision:** Cost volume construction requires precise camera calibration. Errors in extrinsics can lead to incorrect depth estimates [S2, S2, unofficial].

## Operating characteristics
- **Time Complexity:** Inference time is constant with respect to scene complexity but depends on the number of views $N$ and the resolution of the cost volume. Typically faster than iterative optimization methods [S1].
- **Memory Complexity:** Memory usage scales with the size of the cost volume, which depends on the number of views, image resolution, and depth planes. Not given in the sources.
- **Performance:** On the DTU dataset (Scan105), the method produces plausible geometries with 2 views. Quality progressively improves and converges to low error as views increase from 2 to 8 [S1].
- **Strengths:** Generalizable to unseen scenes; handles arbitrary view counts; smoother surfaces compared to density-based methods like MVSNerf [S1].
- **Limitations:** Performance degrades with very sparse views compared to dense inputs; requires accurate camera calibration [S1][S2, S2, unofficial].
- **When to Use:** When a quick reconstruction from a few calibrated views is needed, and iterative optimization is too slow.
- **When Not to Use:** When single-view reconstruction is required without priors (use PixelSplat) or when dense input is available and iterative optimization is feasible for higher quality.

## Background
The concept of handling arbitrary view counts in generalizable neural rendering was demonstrated in SparseNeuS, which showed progressive improvement from 2 to 8 views on the DTU dataset [S1]. The specific integration of cost volume representations into Gaussian Splatting for sparse view reconstruction was introduced by MVSplat, building on earlier work like PixelNeRF and PixelSplat [S2, S2, unofficial].

## Sources
[S1] SparseNeuS: Fast Generalizable Neural Surface Reconstruction from Sparse Views, https://doi.org/10.1007/978-3-031-19824-3_13, fetched 2026-10-26
[S2] PDF Recent advances in 3D Gaussian splatting - Springer, https://link.springer.com/content/pdf/10.1007/s41095-024-0436-y.pdf, fetched 2026-10-29 (unofficial)
