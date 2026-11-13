# Training Schedule

Also called: NeuS2 Training Details

## Overview
A training schedule defines the temporal progression of hyperparameters, loss functions, and structural updates during the optimization of a radiance field or implicit surface [G]. It solves the problem of slow convergence and instability by decoupling coarse geometric alignment from fine detail refinement [G]. Unlike [Learning Rate Scheduling](2701-learning-rate-scheduling.md), which adjusts only the optimizer's step size, or [Regularization and Priors](6281-regularization-and-priors.md), which adds static penalty terms, a training schedule actively changes the *active set* of losses and the *structural state* of the representation (e.g., enabling densification or freezing updates) over specific iteration intervals [G].

## How it works
The implementation of a training schedule depends on the specific method variant [G]. Below are the canonical schedules for static/dynamic neural implicit surfaces, 3D Gaussian Splatting variants, and photon field networks [G].

### Static and Dynamic Neural Implicit Surfaces (NeuS2)
This schedule optimizes a signed distance field (SDF) and neural radiance field [S1].

**Inputs:**
- Training images and camera poses.
- Target iteration counts: $I_{static} = 15,000$ [S1].
- Dynamic frame counts: $I_{first} = 2,000$ (or 4,000 for complex geometry like Lego) [S1], $I_{subsequent} = 1,100$ [S1].

**Steps:**
1. **Static Reconstruction:**
   - Train for $I_{static}$ iterations.
   - Sample 90% of rays on foreground pixels and 10% on background pixels [S1].
2. **Dynamic Reconstruction (First Frame):**
   - Train the first frame from scratch for $I_{first}$ iterations [S1].
3. **Dynamic Reconstruction (Subsequent Frames):**
   - For each subsequent frame:
     - **Global Transform Phase:** Optimize the global transformation matrix independently for the first 100 iterations [S1].
     - **Joint Fine-tuning:** Optimize network parameters and global deformation together for the remaining $I_{subsequent} - 100$ iterations [S1].

**Parameters:**
- Static iterations: 15,000 [S1].
- Dynamic first frame iterations: 2,000 (4,000 for Lego) [S1].
- Dynamic subsequent frame iterations: 1,100 [S1].
- Global transform optimization iterations: 100 [S1].

### 3D Surface Splatting (3DSS)
This schedule manages a set of surfels (surface elements) through three distinct phases [S4, unofficial].

**Inputs:**
- Initial point cloud (e.g., 50k surfels) [S4, unofficial].
- Ground-truth masks (optional) [S4, unofficial].

**Steps:**
1. **Warm-up Phase:**
   - Activate only photometric and silhouette losses [S4, unofficial].
   - Allow the surfel set to deform toward observed images under sparse initial coverage [S4, unofficial].
2. **Densification Phase:**
   - Enable adaptive density control (splitting and pruning) [S4, unofficial].
   - Disable silhouette loss [S4, unofficial].
   - Activate geometric regularizers ($L_{cons}$, $L_{nc}$) [S4, unofficial].
   - **Periodic Pruning:** If ground-truth masks are available, periodically remove floater surfels that consistently project outside the foreground mask across a majority of training views [S4, unofficial].
   - **Densification Frequency:** Perform densification every 4 epochs [S2, unofficial].
   - **Duration:** Continue until 14–25% of the training process is complete [S2, unofficial].
3. **Refinement Phase:**
   - Freeze densification operations [S4, unofficial].
   - Fine-tune with all remaining losses until convergence [S4, unofficial].

**Parameters:**
- Densification interval: Every 4 epochs [S2, unofficial].
- Densification end: 14–25% of total training [S2, unofficial].
- BigBrain Cerebellar Exception: Pruning performed three times: twice during densification at epochs 200 and 300 (ratio 0.8), and once after densification at epoch 800 (ratio 0.85) [S2, unofficial].

### Staggered KNN Training Schedule (Photon Fields)
Used to improve convergence time for KNN-based photon field estimation [S5].

**Inputs:**
- Sequence of increasing KNN query radii $r_i$ approaching a target radius $r$ [S5].
- Total training steps $T$ (typically 2,000–3,000) [S5].

**Steps:**
1. Initialize training step $t = 0$.
2. Determine current sampling radius $r_t$ from the predefined sequence based on $t/T$ [S5].
3. Query KNN with radius $r_t$.
   - If $r_t$ is small, return $k < K$ samples [S5].
   - As $r_t$ increases, $k$ approaches $K$ [S5].
4. Update model parameters using the Relative Mean Square Error (rMSE) loss applied to each set of outputs individually [S5].
5. Increment $t$. Repeat until $t = T$.

**Parameters:**
- Typical duration: 2,000–3,000 steps [S5].
- Optimizer: Adam ($\beta_1 = 0.9, \beta_2 = 0.99$, no L2 regularization) [S5].
- Learning Rate: $9 \times 10^{-4}$ with exponential decay of 0.92 starting at 70% of total steps, triggering every 25 steps [S5].
- Example Schedule (1M Photons) [S5]:
  - 0–36% steps: Radius 25
  - 37–63% steps: Radius 50
  - 64–90% steps: Radius 50
  - 91–100% steps: Radius 0 (target)

### Temporal Sensitivity Sampling (SpeeDe3DGS)
Used to suppress floaters (unstable Gaussians) in dynamic 3D Gaussian Splatting [S6, unofficial].

**Inputs:**
- Deformation network $\mathcal{D}$
- Current timestep $t$
- Noise scaling factor $\beta$
- Timestep interval $\Delta t$
- Current iteration $i$
- Total iterations $\tau$

**Steps:**
1. During sensitivity estimation, perturb the input timestamp to the deformation field [S6, unofficial].
2. Calculate noise magnitude: $\sigma_{noise} = \beta \Delta t (1 - i/\tau)$ [S6, unofficial].
3. Sample noise $\epsilon \sim \mathcal{N}(0,1)$ [S6, unofficial].
4. Compute perturbed timestamp: $t' = t + \epsilon \sigma_{noise}$ [S6, unofficial].
5. Pass $(\mu, r, s, t')$ to $\mathcal{D}$ to obtain deformed Gaussian parameters [S6, unofficial].

**Formula:**
$$ (\mu + \Delta\mu, r + \Delta r, s + \Delta s) = \mathcal{D}(\mu, r, s, t + \mathcal{N}(0,1)\beta\Delta t(1 - i/\tau)) $$

**Pitfall:** Floaters are unstable Gaussians that appear static in observed views but drift at unseen timestamps. TSS reveals them early in training via timestamp jittering [S6, unofficial].

### Update Schedule for Proposal Networks (Ganerf)
Clips the update frequency of proposal networks to ensure stability during warmup [S8, unofficial].

**Inputs:**
- Current step `step`
- Warmup steps `proposal_warmup`
- Max update frequency `proposal_update_every`

**Steps:**
1. Interpolate update frequency linearly from 0 to `proposal_update_every` over the range `[0, proposal_warmup]` [S8, unofficial].
2. Clip the result to `[1, proposal_update_every]` [S8, unofficial].

**Formula:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def get_update_freq(step, proposal_warmup, proposal_update_every):
    if step <= proposal_warmup:
        freq = (step / proposal_warmup) * proposal_update_every
    else:
        freq = proposal_update_every
    return int(max(1, freq))
```
*Note: The source formula `clip(interpolate(...), 1, max_freq)` implies a minimum frequency of 1 even during warmup, which contradicts the text "not updated" [S8, unofficial]. The implementation follows the explicit formula.*

### Coarse-to-Fine 3D Schedule
Controls the kernel parameter $\sigma$ of the Gaussian kernel applied to the 3D radiance field [S9, unofficial].

**Steps:**
1. Exponentially reduce $\sigma$ to 0 over 10,000 iterations [S9, unofficial].
2. Keep $\sigma = 0$ for all subsequent iterations [S9, unofficial].

### Progressive Image Acquisition Schedule (LiTe-GS)
Dynamically increases the number of training views [S10, unofficial].

**Inputs:**
- Initial image count $N_{init}$
- Increment $K$
- Target image count $N_{target}$
- Iterations per stage $I_{stage}$

**Steps:**
1. Initialize training with $N_{init}$ images [S10, unofficial].
2. For each acquisition step:
   - Add $K$ images to the training set.
   - Train for $I_{stage}$ iterations.
3. Repeat until $N_{target}$ images are reached [S10, unofficial].

**Parameters:**
- Standard: $N_{init}=3$, $K \in \{3,4,6\}$, $N_{target}=15$, $I_{stage}=300$, Total $I=15,000$ [S10, unofficial].
- MipNeRF 360 (Garden): $N_{init}=2$, $K=3$, $N_{target}=11$, $I_{stage}=300$, Total $I=78,000$ [S10, unofficial].

## Subtopics

### [Hierarchical Spherical Harmonics Optimization](5444-hierarchical-spherical-harmonics-optimization.md)
Hierarchical Spherical Harmonics (SH) Optimization is a training schedule strategy that iteratively refines a scene representation by optimizing SH coefficients sequentially from a starting band up to a target maximum band. It initializes parameters for each band in order, allowing low-frequency color and scene density to stabilize before introducing higher-frequency details. Choose this when angular resolution is the bottleneck and low-frequency components need to converge first. See [Hierarchical Spherical Harmonics Optimization](5444-hierarchical-spherical-harmonics-optimization.md).

### [Coarse-to-Fine Model Initialization](6793-coarse-to-fine-model-initialization.md)
Coarse-to-Fine Model Initialization accelerates the convergence of dynamic radiance field optimization by leveraging a pre-trained coarse model to initialize a higher-resolution fine model. It addresses the slow convergence and instability associated with training high-resolution deformation fields from scratch. Unlike [Hierarchical Spherical Harmonics Optimization](5444-hierarchical-spherical-harmonics-optimization.md), which refines angular resolution, this method refines spatial resolution and deformation complexity by transferring learned features from a low-resolution or simplified representation to a detailed one. See [Coarse-to-Fine Model Initialization](6793-coarse-to-fine-model-initialization.md).

### Staggered KNN Training Schedule
A training schedule that uses a sequence of increasing KNN query radii $r_i$ approaching a target radius $r$ to improve convergence time. Early stages use small radii for quick adaptation; later stages use larger radii for detail refinement. It provides approximately 2–3× improvement in training speed compared to naïve training using the target radius directly [S5].

### Temporal Sensitivity Sampling
TSS is a technique to suppress floaters (unstable Gaussians) and improve temporal coherence by perturbing timestamps during sensitivity estimation. It reveals unstable primitives early in training via timestamp jittering, with noise magnitude annealing as training progresses [S6, unofficial].

### Evaluation Setup & Metrics
A standardized benchmark protocol for testing differentiable rendering optimization across geometric and non-geometric parameters. It involves collecting 8 scenes with varying parameters (translation, rotation, vertex position, camera pose, material, environment light, textures) and optimizing for 1,000 iterations per run using 128×128 resolution [S7].

### Update Schedule for Proposal Networks
A schedule function that clips the update frequency of proposal networks based on the training step, ensuring updates happen at least once every `proposal_update_every` steps after a warmup period. It interpolates the update frequency linearly from 0 to `proposal_update_every` over the warmup range [S8, unofficial].

### Coarse-to-Fine 3D Schedule
A training schedule that controls the kernel parameter ($\sigma$) of the Gaussian kernel applied to the 3D radiance field, exponentially reducing it to 0 to stabilize optimization. It is applied over 10,000 iterations [S9, unofficial].

### Progressive Image Acquisition Schedule
A schedule that initializes training with a small number of images and adds $K$ images at each acquisition step until reaching a target count. Each stage uses a fixed number of training iterations (e.g., 300), totaling 15,000 iterations for standard setups [S10, unofficial].

### TexGS-VolVis Training Configuration
A configuration schedule for optimizing textured Gaussian splatting attributes. It renders 162 multi-view images using icosphere sampling and applies different learning rates per attribute: 0.01 for shading attributes and `cpal`, 0.025 for texture attributes, and the standard 2DGS rate for others. Optimized using Adam on an NVIDIA RTX4090 [S11].

## Operating characteristics
- **Convergence Speed:** Staggered KNN schedules provide 2–3× faster convergence than naïve approaches [S5].
- **Stability:** The three-phase schedule (Warm-up, Densification, Refinement) in 3DSS prevents instability from adaptive density control during early, sparse coverage [S4, unofficial]. Temporal Sensitivity Sampling suppresses floaters that cause temporal incoherence [S6, unofficial].
- **Complexity:** Progressive Image Acquisition increases oracle evaluation complexity if not managed, but LiTe-GS maintains nearly constant oracle complexity regardless of subset size $K$ [S10, unofficial].
- **Hardware/Resolution:** Static training takes ~5 minutes for 15k iterations [S1]. Dynamic first frame takes ~40 seconds for 2k iterations [S1]. Evaluation typically uses 128×128 resolution [S7].

## Background
Training schedules for radiance fields have evolved from simple fixed-step optimization to complex, multi-phase strategies that manage structural changes (densification, pruning) and loss weighting. NeuS2 introduced specific iteration counts for static and dynamic scenes [S1]. 3DSS formalized the warm-up/densification/refinement phases for surface splatting [S4, unofficial]. Staggered KNN schedules were developed to mitigate the computational cost of large KNN queries in photon mapping [S5]. Temporal Sensitivity Sampling was introduced in SpeeDe3DGS to address floater artifacts in dynamic Gaussian Splatting [S6, unofficial].

## Sources
[S1] NeuS2: Fast Learning of Neural Implicit Surfaces for Multi-view Reconstruction, https://doi.org/10.1109/iccv51070.2023.00305, fetched 2026-10-26
[S2] ESVR: 3D Ellipsoid-based Sparse Volume Rendering via Structure-aware Primitive Learning and Per-primitive Ray Sampling, https://arxiv.org/abs/2608.05564v1, fetched 2026-10-26 (unofficial)
[S3] TRIPS: Trilinear Point Splatting for Real‐Time Radiance Field Rendering, https://doi.org/10.1111/cgf.15012, fetched 2026-10-27
[S4] 3DSS: 3D Surface Splatting for Inverse Rendering, https://doi.org/10.48550/arxiv.2605.05876, fetched 2026-10-26 (unofficial)
[S5] Photon Field Networks for Dynamic Real-Time Volumetric Global Illumination, https://doi.org/10.1109/tvcg.2023.3327107, fetched 2026-10-26
[S6] SpeeDe3DGS: Speedy Deformable 3D Gaussian Splatting with Temporal Pruning and Motion Grouping, https://speede3dgs.github.io/, fetched 2026-10-29 (unofficial)
[S7] Differentiable Rendering Using RGBXY Derivatives and Optimal Transport, https://doi.org/10.1145/3550454.3555479, fetched 2026-10-27
[S8] barbararoessle/ganerf, https://github.com/barbararoessle/ganerf, fetched 2026-10-27 (unofficial)
[S9] Improving Robustness for Joint Optimization of Camera Poses and Decomposed Low-Rank Tensorial Radiance Fields, https://doi.org/10.48550/arxiv.2402.13252, fetched 2026-10-26 (unofficial)
[S10] LiTe-GS: Oracle-Efficient Next Best View Selection for 3D Gaussian Splatting, https://arxiv.org/abs/2609.30393, fetched 2026-10-29 (unofficial)
[S11] TexGS-VolVis: Expressive Scene Editing for Volume Visualization via Textured Gaussian Splatting., https://doi.org/10.1109/tvcg.2025.3634643, fetched 2026-10-30
