# Depth-Guided Sampling

Also called: Depth-Guided Probability Fields, Gaussian Boosting

## Overview
Depth-guided sampling is a ray sampling strategy that focuses scene sampling on estimated surface regions using depth priors from input views to increase sample density in relevant areas, thereby improving synthesis quality and allowing a reduction in samples per ray [S1, S2]. It differs from standard uniform sampling by leveraging predicted depth maps to define probability density fields for surface presence, and from simple depth-clipping by incorporating occlusion-aware likelihoods and Gaussian boosting to refine sample placement around the termination expectation [S1].

## How it works
**Inputs and Outputs:**
*   **Inputs:** Ray $r(t)$ defined by near plane $t_{near}$ and far plane $t_{far}$ [S1]. Per-view depth expectation maps $D^{(i)}$ and standard deviation maps $D_{std}^{(i)}$ for each input view $i$ [S1]. Camera projection matrices $K^{(i)}$ [S1]. Parameters: $N_{cand}$ (number of candidate samples), $N_{samples}$ (number of shortlisted samples), $N_{gauss}$ (number of Gaussian samples), $\delta$ (step size) [S1].
*   **Outputs:** A set of query points $\{x\}$ along the ray, optimized for surface presence [S1].

**Data Structures:**
*   **Candidate Samples:** A list of $N_{cand}$ points uniformly distributed along the ray segment $[t_{near}, t_{far}]$ [S1].
*   **Likelihoods:** Scalar values $p^{(i)}(x)$ representing the probability of a point $x$ being a surface point for view $i$, and $p(x)$ representing the multi-view likelihood [S1].
*   **Occlusion-Aware Likelihoods:** Scalar values $p_{oa}(x)$ representing the likelihood of a ray terminating at $x$ [S1].

**Steps:**

1.  **Candidate Sampling:** Uniformly sample $N_{cand}$ candidate points $\{x\}$ along the ray $r(t)$ within the interval $[t_{near}, t_{far}]$ [S1].
    $$ \{x\} = \{r(t) \mid t \sim [t_{near}, t_{far}]\} $$
    The step size $\delta$ for these candidates is defined as:
    $$ \delta = (t_{far} - t_{near}) / N_{cand} $$
    [S1]

2.  **Backface Culling:** For each input view $i$, calculate surface normals from the depth map $D^{(i)}$ [S1]. Discard candidate samples if the angle between the ray direction and the projected normal value is smaller than $90^\circ$ [S1].

3.  **Per-View Likelihood Calculation:** For each candidate point $x$ and each input view $i$:
    a. Calculate the difference $\Delta z$ between the sample's z-component in the $i$-th camera space and its corresponding projected depth value:
    $$ \Delta z = D^{(i)}(K^{(i)} \circ x^{(i)}) - x[z] $$
    where $x^{(i)}$ is the point $x$ in camera $i$'s coordinate system, $x[z]$ is its z-component, and $K^{(i)} \circ x^{(i)}$ denotes the projection of $x$ onto the image plane of view $i$ [S1].
    b. Determine the mean $\mu$ and standard deviation $\sigma$ for the Gaussian PDF using the depth estimator outputs:
    $$ \mu = D^{(i)}(K^{(i)} \circ x^{(i)}) $$
    $$ \sigma = D_{std}^{(i)}(K^{(i)} \circ x^{(i)}) $$
    [S1]
    c. Calculate the likelihood $p^{(i)}(x)$ of $x$ being a surface point for view $i$ by integrating a Gaussian PDF over the sample interval $\delta$:
    $$ p^{(i)}(x) = \int_{z-\delta/2}^{z+\delta/2} \frac{1}{\sqrt{2\pi}\sigma} e^{-\frac{1}{2} \left(\frac{z'-\mu}{\sigma}\right)^2} dz' $$
    where $z = x[z]$ [S1].

4.  **Multi-View Likelihood Merging:** Determine the multi-view likelihood $p(x)$ by view-wise max-pooling:
    $$ p(x) = \max_i p^{(i)}(x) $$
    [S1]

5.  **Shortlisting:** Shortlist $N_{samples}$ candidates with the highest likelihoods $p(x)$ for sampling the NeRF [S1].

6.  **Gaussian Boosting:**
    a. Calculate the occlusion-aware likelihood $p_{oa}(x)$ for each candidate sample $x$ along the ray $r$:
    $$ p_{oa}(x) = p(x) \cdot \prod_{\tilde{x} < x} (1 - p(\tilde{x})) $$
    where the product is taken over all candidate samples $\tilde{x}$ satisfying $\tilde{x} < x$ along the ray parameter [S1].
    b. Fit a Gaussian distribution against the occlusion-aware likelihoods $p_{oa}(x)$ along the ray [S1].
    c. Sample $N_{gauss}$ points from this fitted Gaussian distribution [S1].
    d. Add these $N_{gauss}$ points to the shortlisted candidates [S1].

**Alternative Strategy (Uniform Depth Interval):**
Some implementations simplify the process by uniformly sampling $M$ points within the confidence interval $[d-s, d+s]$ of the estimated depth $d$, skipping coarse sampling [S2]. The depth $t_j$ of the $j$-th query point ($j=1..M$) is calculated as:
$$ t_j = d - s + \frac{2s}{M - 1} (j - 1) $$
where $d$ is the estimated depth and $s$ is the confidence interval [S2].

**Parameters:**
*   $N_{cand}$: Number of initial candidate samples. Not given in the sources.
*   $N_{samples}$: Number of candidates shortlisted based on likelihood. Not given in the sources.
*   $N_{gauss}$: Number of additional points sampled via Gaussian boosting. Not given in the sources.
*   $\delta$: Step size for candidate samples, defined as $(t_{far} - t_{near})/N_{cand}$ [S1].
*   $M$: Number of points in uniform depth interval sampling. Not given in the sources.
*   $d$: Estimated depth. Not given in the sources.
*   $s$: Confidence interval. Not given in the sources.

**Edge Cases and Pitfalls:**
*   If the confidence interval $s$ is large, sampling might still cover blank areas, although the method reduces this compared to uniform camera-space sampling [S2].
*   Backface culling requires accurate normal estimation from depth maps; errors here may lead to incorrect discarding of valid surface samples [S1].
*   The integration of the Gaussian PDF in step 3c may require numerical approximation if closed-form solutions are not used, though the sources present the integral form [S1].

## Subtopics

### Stereo Depth-Cueing Ray Marching
A sampling strategy that uses stereo depth priors to concentrate point sampling near tissue surfaces, replacing hierarchical stratified sampling [S3]. It discretizes ray $r(s)$ into points $\{x_j \mid x_j = r(s_j)\}$ for $j=1..m$ by sampling ray steps $s_1 \le s_2 \le \dots \le s_m$ [S3]. Steps $s_j$ are drawn from the normalized impulse distribution $\frac{1}{\xi\sqrt{2\pi}} \delta(s; u, v, i)$ [S3]. The Gaussian transfer function is formulated as $\delta(s; u, v, i) = \exp(-(s - D_i[u, v])^2 / 2\xi^2)$ [S3]. This creates an impulse distribution centered at depth $D_i[u, v]$ [S3]. $D_i[u, v]$ is the coarse depth map for frame $i$ at pixel $(u, v)$, estimated from binocular captures [S3]. $\xi$ is a hyperparameter controlling the width of the Gaussian impulse, set to a small value to mimic a Dirac delta impulse [S3].

## Operating characteristics
*   **Time Complexity:** Not given in the sources.
*   **Memory Complexity:** Not given in the sources.
*   **Measured Performance:** Validated on the FaceScape dataset [S1]. The method allows reduction of samples per ray while maintaining or improving synthesis quality [S1, S2].
*   **Strengths:** Focuses sampling on relevant surface regions, reducing wasted samples in empty space [S1, S2].
*   **Limitations:** Relies on the accuracy of the depth estimator [S1, S2].

## Background
The method was introduced in the DINER paper (Depth-aware Image-based NEural Radiance fields) [S1]. It builds on pixelNeRF and leverages the TransMVSNet architecture for depth estimation [S1]. The approach exploits depth from input views that do not coincide with the target view, predicting depth at both training and test time [S1]. A related approach, NeRF-SDP, also uses depth perception for efficient generalizable NeRFs, employing a depth-encoded ray transformer and uniform sampling within confidence intervals [S2].

## Sources
[S1] DINER: Depth-aware Image-based NEural Radiance fields, https://doi.org/10.1109/cvpr52729.2023.01198, fetched 2026-10-29
[S2] NeRF-SDP: Efficient Generalizable Neural Radiance Field with Scene Depth Perception, https://doi.org/10.1145/3595916.3626380, fetched 2026-10-30
[S3] Neural Rendering for Stereo 3D Reconstruction of Deformable Tissues in Robotic Surgery, https://doi.org/10.1007/978-3-031-16449-1_41, fetched 2026-10-26
