# Evaluation and Reconstruction Metrics
Also called: Evaluation Methodology for SLF
## Overview
This family provides metrics and objectives to evaluate the quality, stability, and robustness of radiance field optimizations and volume renderings, distinct from the regularization losses that drive optimization [G]. It quantifies final output fidelity to ground truth, temporal coherence, geometric consistency, or robustness to perturbations [G]. Unlike sibling methods that enforce smoothness or perceptual similarity, these metrics serve as diagnostic tools or final benchmarks to assess specific failure modes such as flickering, geometric inconsistency across views, or adversarial sensitivity [G].
## Shared principles

## Subtopics

### [Temporal Instability Metric](1921-temporal-instability-metric.md)
Quantifies temporal coherence by measuring frame-to-frame variations in reconstruction error. Lower values indicate smoother temporal consistency.

### [Inverse Mean Residual Color (IMRC) Metric](10556-inverse-mean-residual-color-imrc-metric.md)
Evaluates geometric quality without ground-truth geometry by measuring transmittance-density-weighted residual color in decibels. Evaluates the entire density field.

### [Across View-direction Inconsistency (AVI) Metric](11106-across-view-direction-inconsistency-avi-metric.md)
Quantifies geometrical inconsistency between adjacent viewing directions by leveraging ground truth optical flow to warp views. Targets 3D geometry consistency.

### [Ray Diversity Objective](13003-ray-diversity-objective.md)
Quantifies ray variation from various poses observing a mesh face. Combines geometric diversity with a quality penalty to guide active view planning.

### [Adversarial Robustness Evaluation](16421-adversarial-robustness-evaluation.md)
Assesses stability under small input perturbations by computing the BREN ratio. Evaluates sensitivity to artifacts.

### [Ambiguity-Aware Inverse Rendering Evaluation Metrics](19381-ambiguity-aware-inverse-rendering-evaluation-metrics.md)
Quantifies uncertainty by measuring variance across stochastic samples. Evaluates stability under varying conditions.

### [Registration Metric Evaluation Protocol](21625-registration-metric-evaluation-protocol.md)
Optimization-independent method evaluating rigid registration similarity by sampling the parameter space around a golden standard. Provides statistical characterizations of the metric's landscape.

### [Reconstruction Error (RE) Metric](21653-reconstruction-error-re-metric.md)
Quantifies pixel-wise discrepancy using the L2 norm. Provides a direct measure of photometric fidelity.

### Leak Metric
Measures the fraction of a method's near-plane foreground energy that incorrectly falls on the culled side where the reference is background. Combines proxy reconstruction error, renderer mismatch, soft tails, and overshooting culls [S2, unofficial]. Steps: 1. Compute method foreground mask $F(R)$ using $\theta_{fg} = 0.04$. 2. Define near-plane region $N$ as pixels in $F(R)$ with $|s| \leq 60$. 3. Define leak region $L$ as pixels in $N$ with $s > 2$ that are not in reference foreground $F(G)$. 4. Sum $\max_c R_c(\mathbf{x})$ over $L$ and $N$ across all grazing views, then take ratio [S2, unofficial]. Formula: $N = F(R) \cap \{|s| \leq 60\}$, $L = N \cap \{s > 2\} \setminus F(G)$. $\text{Leak} = \frac{\sum_{\mathbf{x} \in \mathcal{L}} \max_{\mathbf{x}} R_c(\mathbf{x})}{\sum_{\mathbf{x} \in \mathcal{N}} \max_{\mathbf{x}} R_c(\mathbf{x})}$ [S2, unofficial]. Parameters: Foreground threshold $\theta_{fg} = 0.04$. Cut-edge margin $\delta = 2$ (for $s > 2$). Near-plane half-width $w = 60$ (for $|s| \leq 60$) [S2, unofficial]. Pitfalls: Averaging per-view ratios on thin grazing bands causes numerical instability. Pooling sums across all views before taking the ratio stabilizes the metric [S2, unofficial].

### Band Fidelity Metric (Sb)
A fidelity metric measuring the structural similarity between a method's render and a reference render over the exposed cut face (reference foreground) [S2, unofficial]. Steps: Compute scalar luminance $Y(x) = \frac{1}{3} \sum_c I_c(x) \in [0, 1]$ at each pixel $x$. Compute foreground mask $F(I) = \{x : \max_c I_c(x) > \theta_{fg}\}$ with $\theta_{fg} = 0.04$. Compute SSIM between method render and reference restricted to $F(G)$. Average the SSIM values across all perpendicular views of all planes in the evaluated set [S2, unofficial]. Parameters: Foreground threshold $\theta_{fg} = 0.04$. Band $\mathcal{B}$ for $S_b$ is the reference foreground $F(G)$ [S2, unofficial]. Pitfalls: Band PSNR is omitted from main tables because operator differences are only hundredths of a decibel and leakage can offset small brightness bias [S2, unofficial].

### Model Validation using RMSE and EMD
A validation procedure assessing model generalization on a hold-out set of 2,000 volume-rendered images not used for training [S3]. EMD (Earth Mover’s Distance) is used to measure higher-level similarity between color histograms, mitigating misalignment in histogram space [S3]. Parameters: RMSE (Root Mean Squared Error) is used as the primary evaluation metric for image similarity [S3]. EMD Cost Normalization: The cost between histogram bins is normalized such that the maximum cost is 1 [S3]. Numbers: Training Time: 16.5 hours. Rendering Time: 0.007 seconds per image. TF Exploration Time: 0.06 seconds. TF Sensitivity Time: 0.49 seconds [S3].

### Baseline Calibration
A calibration procedure that drives a scene reconstruction as close as possible to ground truth in image space without using network losses, to establish a threshold for metamerism [S4, unofficial]. Steps: For shape reconstruction, use mean-absolute error between rendered and ground-truth views. For material reconstruction, use the Dual-Buffer loss to handle Monte Carlo noise from path tracing. Compute the baseline score using the network's latent space projection on masked images [S4, unofficial]. Formula: $\text{score}_{\text{baseline}} = \text{score}(\text{NN}(m(n) \odot \hat{x}(n)), \text{NN}(m(n) \odot x_i(n)))$ [S4, unofficial]. Parameters: $m(n)$: A binary foreground mask that suppresses background regions [S4, unofficial]. Pitfalls: Do not mask material experiments because the object's appearance depends on global illumination and indirect lighting; masking would discard appearance cues [S4, unofficial].

## Background
Evaluation metrics for volume rendering and radiance fields have evolved alongside the complexity of reconstruction methods. Early metrics focused on simple pixel-wise errors like PSNR and SSIM [G]. Specialized metrics have been developed to address specific artifacts and evaluation needs, such as temporal instability in novel view synthesis [G], geometric consistency across views [G], and geometry evaluation without ground-truth meshes [G]. The family also includes specialized metrics for evaluating clipping and cutting operations in medical volume rendering, such as Leak and Band Fidelity, which address artifacts specific to half-space clipping techniques [S2, unofficial].
## Sources
[S1] Surface Light Field Compression Using a Point Cloud Codec, https://doi.org/10.1109/jetcas.2018.2883479, fetched 2026-10-28
[S2] XClipGS: Exact Half-Space Clipping for Medical Volume Gaussian Splatting, https://doi.org/10.48550/arxiv.2608.07760, fetched 2026-10-26 (unofficial)
[S3] A Generative Model for Volume Rendering, https://doi.org/10.1109/tvcg.2018.2816059, fetched 2026-10-26
[S4] MRD: Using Physically Based Differentiable Rendering to Probe Vision Models for 3D Scene Understanding, https://doi.org/10.48550/arXiv.2512.12307, fetched 2026-10-29 (unofficial)
