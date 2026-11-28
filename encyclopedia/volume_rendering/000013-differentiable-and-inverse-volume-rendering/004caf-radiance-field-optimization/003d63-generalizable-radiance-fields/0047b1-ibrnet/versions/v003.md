# IBRNet

Also called: Instant Neural Graphics Primitives with Image-Based Rendering Networks

## Overview
IBRNet is a feedforward neural network method for novel view synthesis that generalizes to unseen scenes without per-scene optimization [S1]. It differs from standard NeRF and siblings like [NeRF-SDP](8554-nerf-sdp.md) by replacing iterative volume rendering optimization with a CNN-based feature extraction and a Ray Transformer that predicts density directly from multi-view features [S1]. It builds on [Generalizable Radiance Fields](9480-generalizable-radiance-fields.md) by using a data prior learned from a large dataset to bypass scene-specific training [S2, unofficial].

## How it works
**Inputs and Outputs:**
- Input: A target camera pose and $N$ source images from the training set with similar viewing directions [S1].
- Output: Density $\sigma$ for query points along rays from the target camera [S1].

**Data Structures:**
- Feature Volume: Implicitly constructed via feature aggregation [S1]. No explicit 3D grid is stored; features are projected from 2D image features [S1].
- Ray Transformer: A sequence model processing concatenated features along a ray [S1].

**Steps:**
1. **View Selection:** For a target view, select $N$ source views from the training set with the most similar viewing directions [S1].
2. **Feature Extraction:** Use a CNN to extract feature maps $F_i$ from each of the $N$ source images [S1].
3. **Query Point Projection:** For a query point $x$ along a ray from the target camera, project $x$ onto each source image $i$ using the known camera matrix $K_i$ to obtain pixel coordinates [S1].
4. **Feature Sampling:** Sample the color $c_i$ and feature vector $f_i$ from the CNN features at the projected coordinates for each view $i$ [S1].
5. **Feature Refinement and Weighting:** Pass each feature $f_i$ through an MLP to produce a refined multi-view aware feature $f'_i$ and a pooling weight $w_i$ [S1].
6. **Feature Aggregation:** Compute the aggregated feature $F_{agg}$ for point $x$ by summing the refined features weighted by $w_i$:
   $$ F_{agg}(x) = \sum_{i=1}^{N} w_i f'_i $$
   This step is repeated for all query points along the ray [S1].
7. **Ray Transformer Processing:** Concatenate the aggregated features $F_{agg}(x)$ for all query points along the ray into a sequence [S1]. Feed this sequence into a Ray Transformer [S1].
8. **Density Prediction:** The Ray Transformer outputs the density $\sigma$ for each query point [S1]. The source emphasizes density prediction by the transformer [S1]. Color prediction details are not provided in the sources.

**Parameters:**
- $N$: Number of source views. Not given in the sources.
- CNN Backbone: Not given in the sources.
- MLP Dimensions: Not given in the sources.

**Edge Cases and Pitfalls:**
- View Selection: The method selects $N$ source views with the most similar viewing directions [S1].
- Precision: Projection accuracy depends on camera calibration [S1].

## Subtopics

### NaN
A denoising method that incorporates inter-view and spatial awareness to enhance noise robustness, achieving state-of-the-art results in burst denoising under challenging conditions like large motion and high noise [S1]. It avoids per-scene training by building on IBRNet, which generalizes to unseen scenes with minimal input [S1].

## Operating characteristics
**Time and Memory Complexity:**
- Time: Feedforward inference, significantly faster than iterative NeRF optimization [S2, unofficial].
- Memory: Depends on the number of source views $N$ and the CNN feature size. No explicit 3D volume storage required [S1].

**Measured Performance:**
- Widely used in benchmarks for view synthesis [S1].
- Reduces the number of iterations compared to per-scene NeRF [S3, unofficial].

**Strengths and Limitations:**
- Strengths: Generalizes to unseen scenes without per-scene optimization; fast inference [S1][S2, unofficial].
- Limitations: Requires a training dataset; performance depends on the availability of similar source views [S1].

**When to Use:**
- Use for novel view synthesis where per-scene optimization is too slow [S2, unofficial].
- Not suitable for scenes with no similar training views [S1].

## Background
IBRNet was published in February 2021 as a NeRF-adjacent method for view synthesis [S1]. It was introduced to address the slow training speed of NeRF by using a feedforward model to learn a data prior [S3, unofficial]. It is widely used in benchmarks [S1].

## Sources
[S1] Neural Radiance Fields in 3D Vision: A Comprehensive Review, https://doi.org/10.26599/cvm.2026.9450535, fetched 2026-10-28
[S2] Fast Training of Neural Lumigraph Representations using Meta Learning, https://doi.org/10.48550/arxiv.2106.14942, fetched 2026-10-26 (unofficial)
[S3] NeRF的基础及后续扩展, https://pengsida.net/files/NeRF_Tutorial.pdf, fetched 2026-10-30 (unofficial)
