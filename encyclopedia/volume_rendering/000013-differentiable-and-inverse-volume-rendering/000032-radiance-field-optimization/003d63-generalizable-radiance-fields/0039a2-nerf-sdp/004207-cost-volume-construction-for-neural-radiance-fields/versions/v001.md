# Cost Volume Construction for Neural Radiance Fields

Also called: Homographic warping cost volume, Variance-based cost volume

## Overview
This method constructs a 3D cost volume from multiple input views to enable generalizable neural radiance fields (NeRFs) by encoding multi-view consistency. It solves the problem of inferring scene geometry and appearance without per-scene optimization by warping 2D features from input views to a reference view across a range of depths and aggregating their variance. It differs from standard NeRFs by replacing per-ray MLP optimization with a differentiable 3D convolutional encoding of a pre-computed cost volume, and from other generalizable methods by explicitly using homographic warping for feature alignment rather than simple depth projection or attention mechanisms.

## How it works
**Inputs:**
- $N$ input views, each an image $I_i \in \mathbb{R}^{H_i \times W_i \times 3}$ for $i=1 \dots N$.
- Camera parameters for each view $i$: intrinsic matrix $K_i$, rotation matrix $R_i$, translation vector $t_i$.
- A reference view (typically view 1) with parameters $K_1, R_1, t_1$.
- A set of discrete depth values $g$ defining the scanning volume.
- Hyperparameters: feature channel count $C$, number of input views $N$.

**Outputs:**
- Cost volume $P \in \mathbb{R}^{H_{ref}/4 \times W_{ref}/4 \times D \times C}$, where $D$ is the number of depth samples.
- Neural feature volume $S = B(P)$, where $B$ is a 3D CNN (see Subtopics).

**Data Structures:**
- Feature maps $F_i$: Extracted via CNN, layout $\mathbb{R}^{H_i/4 \times W_i/4 \times C}$.
- Warped feature maps $F_{i,g}$: Same layout as $F_i$, aligned to reference view coordinates.
- Cost volume $P$: 4D tensor. Dimensions correspond to reference view spatial resolution (downsampled by 4), depth steps, and feature channels.

**Steps:**

1. **Feature Extraction:**
   For each input view $I_i$, pass through a deep CNN with down-sampling convolutional layers to extract a high-dimensional feature map $F_i$.
   $$ F_i \in \mathbb{R}^{H_i/4 \times W_i/4 \times C} $$
   [G] The CNN typically uses stride-2 convolutions to achieve the 1/4 resolution reduction.

2. **Homographic Warping:**
   For each view $i$ and each depth sample $g$:
   a. Compute the homographic deformation matrix $Q_i(g)$ that maps points from the reference view to view $i$ at depth $g$. The formula is:
   $$ Q_i(g) = K_i R_i R_1^T + \frac{(t_1 - t_i)n}{n_1} 1^T K_1^{-1} $$
   Where:
   - $K_i, K_1$: Intrinsic matrices.
   - $R_i, R_1$: Rotation matrices.
   - $t_i, t_1$: Translation vectors.
   - $n$: Normal vector of the plane at depth $g$ in the reference camera coordinate system.
   - $n_1$: The depth component of $n$ (or the distance to the plane along the reference camera's z-axis, depending on normalization).
   - $1$: A vector of ones.
   
   *Note on $n$ and $n_1$:* In standard homography for planar scenes or depth slices, $n$ is the normal to the plane. For a specific depth $g$ in the reference camera frame (where $Z=g$), the plane equation is $Z=g$. The normal is $n=[0, 0, 1]^T$ and the distance is $g$. However, the source formula uses a general $n$. If implementing for a fixed depth $g$ in the reference frame, $n$ is typically $[0,0,1]^T$ and $n_1 = g$. The term $\frac{(t_1 - t_i)n}{n_1}$ represents the translation component due to baseline.

   b. Warp the feature map $F_i$ to the reference view at depth $g$ using bilinear interpolation. For each pixel $(a, b)$ in the reference view (at 1/4 resolution):
   $$ F_{i,g}(a, b) = F_i(Q_i(g)[a, b, 1]^T) $$
   Here, $Q_i(g)[a, b, 1]^T$ computes the corresponding coordinates in view $i$. Bilinear interpolation is used to sample $F_i$ at these non-integer coordinates.

3. **Cost Volume Aggregation:**
   For each voxel at position $(a, b, g)$ in the cost volume:
   Collect the warped feature vectors from all $N$ views: $\{ F_{1,g}(a, b), F_{2,g}(a, b), \dots, F_{N,g}(a, b) \}$.
   Compute the variance of these feature vectors across the $N$ views to determine the cost feature vector $P(a, b, g)$:
   $$ P(a, b, g) = \text{Var}(F_{i,g}(a, b)) $$
   [G] $\text{Var}(\cdot)$ is computed channel-wise. For each channel $c$, $P(a, b, g)_c = \frac{1}{N} \sum_{i=1}^N (F_{i,g}(a, b)_c - \mu_c)^2$, where $\mu_c$ is the mean across views for channel $c$. This results in a vector of size $C$.

4. **Neural Encoding (Variant):**
   Process the cost volume $P$ with a 3D CNN $B$ (e.g., UNet structure) to produce the neural feature volume $S$:
   $$ S = B(P) $$
   This volume $S$ encodes scene geometry and appearance attributes (lighting, textures, materials) for subsequent density and color prediction.

**Parameters:**
- $C$: Number of feature channels. Typical values not specified in source, but must be consistent across CNN layers.
- $N$: Number of input views.
- Depth range and sampling count $M$: Not explicitly defined in the cost volume section, but implied by the set of $g$ values.

**Edge Cases and Pitfalls:**
- **Occlusions:** Homographic warping assumes a planar scene or consistent depth. Occlusions cause mismatches in warped features, increasing variance incorrectly. The variance metric assumes multi-view consistency; occluded regions may have high variance even if not at the correct depth.
- **Depth Ambiguity:** At large depths or flat textures, variance may be low for incorrect depths, leading to ambiguity.
- **Precision:** Bilinear interpolation during warping introduces smoothing. High-frequency details may be lost.
- **Camera Parameters:** Accurate $K, R, t$ are critical. Errors in extrinsics lead to misalignment and incorrect cost volume.

## Subtopics

### Neural Encoding via 3D CNN
A variant step where the cost volume $P$ is processed by a 3D CNN $B$ with a UNet structure to extract scene geometry and appearance characteristics, producing a neural feature volume $S = B(P)$. This step transforms the raw variance-based cost features into a high-dimensional feature space where channels represent distinct attributes (lighting, textures, materials), enabling inference of volume density and brightness dependent on viewpoint. It sets itself apart by adding a learnable 3D convolutional encoder to the static cost volume, allowing the network to refine geometric and appearance cues before rendering. Choose this when high-fidelity rendering of complex materials and lighting is required, as opposed to using the raw cost volume directly.

## Operating characteristics
- **Time Complexity:** Dominated by feature extraction ($O(N \cdot H \cdot W \cdot C)$) and warping ($O(N \cdot D \cdot H \cdot W \cdot C)$). Warping involves interpolation for each voxel in the volume.
- **Memory Complexity:** $O(N \cdot D \cdot H \cdot W \cdot C)$ to store intermediate warped features or the final cost volume. The cost volume itself is $O(D \cdot H \cdot W \cdot C)$.
- **Performance:** Enables fast novel view synthesis as it avoids per-scene optimization. Performance depends on the accuracy of the homographic warping and the quality of the 3D CNN encoder.
- **Strengths:** Generalizable to unseen scenes; leverages multi-view consistency; differentiable end-to-end.
- **Limitations:** Sensitive to camera calibration errors; struggles with occlusions and non-Lambertian surfaces; resolution limited by the cost volume size and warping accuracy.
- **When to use:** Sparse view synthesis, real-time applications where per-scene optimization is too slow.
- **When not to use:** Scenes with significant occlusions, dynamic content, or when high-precision depth is required without additional regularization.

## Background
Originated in the context of generalizable neural radiance fields, specifically in the work by [S1] (2024/2025). It builds on earlier work in multi-view stereo and homography-based warping [37] (cited in [S1]) and UNet-based 3D CNNs [38] (cited in [S1]). The method addresses the limitation of traditional NeRFs that require per-scene optimization, by introducing a cost volume that encodes multi-view consistency, allowing for generalizable rendering.

## Sources
[S1] Rugularizing generalizable neural radiance field with limited-view images, https://link.springer.com/article/10.1007/s40747-024-01696-6, fetched 2026-10-29
