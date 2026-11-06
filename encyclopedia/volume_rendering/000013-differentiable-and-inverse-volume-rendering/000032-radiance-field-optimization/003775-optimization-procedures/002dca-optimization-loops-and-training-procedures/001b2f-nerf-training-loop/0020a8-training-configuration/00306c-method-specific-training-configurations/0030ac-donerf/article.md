# DONeRF
Also called: DONeRF (Depth Oracle Neural Radiance Fields), DONeRF Training Configuration, DONeRF training setup, Depth Oracle Network

## Overview
DONeRF is a training configuration for a compact dual-network architecture that enables real-time neural rendering with very few samples per ray [S1]. It replaces standard NeRF's dense raymarching with a two-stage pipeline where a depth oracle network predicts sample locations and a shading network accumulates RGBA output [S1]. It differs from [NeRF Hyperparameters](3065-nerf-hyperparameters.md) by enforcing a strict two-phase training schedule and specific opacity regularization to prevent dark pixel artifacts at low sample counts [S1].

## How it works
### Inputs and Outputs
*   **Inputs:** Camera poses, RGB image pixels, and optionally ground truth depth maps [S1].
*   **Outputs:** RGB color values for novel views [S1].

### Data Structures
Both the Depth Oracle Network and the Shading Network share an identical architecture [S1]:
*   **Layers:** 8 hidden layers [S1].
*   **Hidden Units:** 256 units per layer [S1].
*   **Activation:** ReLU [G: Standard for MLPs of this era].
*   **Depth Oracle Output:** Probability distribution over discretized depth bins (classification) [S1].
*   **Shading Network Output:** RGB color and per-sample opacity $\delta$ [S1].

### Training Steps
The method executes a five-stage pipeline during rendering and a two-phase training schedule [S1]:
1.  **Ray Unification:** Ray descriptions are unified within a view cell bounding box [S1].
2.  **Non-linear Depth Space:** Depth is transformed logarithmically to handle large scenes [S1].
3.  **Oracle Classification:** The oracle network learns a classification task on filtered, discretized target depth values rather than predicting a single depth [S1].
4.  **Warping:** Sample positions are warped towards the view cell to direct high-frequency predictions to the foreground [S1].
5.  **Shading Accumulation:** The shading network places a small number of samples guided by the oracle and accumulates RGBA output [S1].

### Formulas
**Opacity Loss ($\text{loss}_O$):**
This term prevents the network from mixing background color via opacity to satisfy the RGB loss, which causes dark pixel artifacts at low sample counts [S1]. It forces accumulated ray opacity to be at least 1 [S1]:
$$
\text{loss}_O = \begin{cases} 
0, & \text{if } \sum_{i=1}^X \delta_i \geq 1 \\
\left( \left( \sum_{i=1}^X \delta_i \right) - 1 \right)^2, & \text{otherwise}
\end{cases}
$$
Where $X$ is the number of samples per ray and $\delta_i$ is the opacity of the $i$-th sample [S1].

**Total Loss:**
The final loss is a weighted sum of the MSE loss and the opacity loss [S1]:
$$
\text{loss} = \alpha \cdot \text{loss}_{\text{MSE}} + \beta \cdot \text{loss}_{\text{O}}
$$
Where $\text{loss}_{\text{MSE}}$ is the Mean Squared Error between predicted and ground truth RGB [S1].

### Parameters
*   **Optimizer:** Adam [S1].
*   **Learning Rate:** $0.0005$ [S1].
*   **Batch Size:** $4096$ samples per iteration [S1].
*   **Training Iterations:** $300,000$ per phase [S1].
*   **Neighborhood Filter Size ($K$):** $5$ [S1].
*   **Depth Smoothing Filter Size ($Z$):** $5$ [S1].
*   **Oracle Input Positions ($I$):** $128$ [S1].
*   **Samples per Ray ($N$):** Variable (e.g., DONeRF-2, DONeRF-4) [S1].
*   **MSE Weight ($\alpha$):** $1.0$ [S1].
*   **Opacity Loss Weight ($\beta$):** $10.0$ [S1].

### Edge Cases and Pitfalls
*   **Dark Pixel Artifacts:** Low sample counts cause rays to fail accumulating to opacity 1 for opaque surfaces, darkening the image [S1]. The opacity loss term remedies this [S1].
*   **Reflective/Transparent Surfaces:** Low sample counts and single depth estimates cannot perfectly replicate view-dependent phenomena like mirrors or refractive water [S1].
*   **No-GT Training Noise:** When ground truth depth is unavailable, a dense NeRF extracts depth maps that may output high-frequency values in the background, creating zero-contribution samples [S1].

### Pseudocode
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import math
import random

# --- Helper Definitions ---
class MLP:
    def __init__(self, num_layers, units):
        self.weights = []
        self.biases = []
        for _ in range(num_layers):
            self.weights.append([[random.gauss(0, 0.1) for _ in range(units)] for _ in range(units)])
            self.biases.append([0.0 for _ in range(units)])
        self.weights.append([[random.gauss(0, 0.1) for _ in range(4)] for _ in range(units)]) # 3 pos + 1 opacity
        self.biases.append([0.0 for _ in range(4)])

    def forward(self, x):
        h = x
        for w, b in zip(self.weights[:-1], self.biases[:-1]):
            h = [max(0.0, sum(wi * xi for wi, xi in zip(w, h)) + bi) for wi, bi in zip(w, b)]
        out = [sum(wi * xi for wi, xi in zip(self.weights[-1], h)) + self.biases[-1][i] for i, wi in enumerate(self.weights[-1])]
        return out

def adam_step(params, grads, lr, t):
    for p, g in zip(params, grads):
        p -= lr * g

def bce_loss(pred, target):
    eps = 1e-7
    pred = max(eps, min(1 - eps, pred))
    return -(target * math.log(pred) + (1 - target) * math.log(1 - pred))

def mse_loss(pred, target):
    return sum((p - t) ** 2 for p, t in zip(pred, target))

def sample_rays(batch_size):
    rays = [(random.uniform(-1, 1), random.uniform(-1, 1)) for _ in range(batch_size)]
    positions = [[random.uniform(0, 1), random.uniform(0, 1), random.uniform(0, 1)] for _ in range(batch_size)]
    return rays, positions

def warp_and_generate_features(positions, rays):
    warped = [[math.log(max(1e-6, p[0] + 1)) for p in pos] for pos in positions]
    features = [[1.0] * 4 for _ in positions]
    return warped, features

# --- Configuration ---
NUM_ITERATIONS = 300000
LEARNING_RATE = 0.0005
BATCH_SIZE = 4096
ALPHA = 1.0
BETA = 10.0
NUM_HIDDEN_LAYERS = 8
HIDDEN_UNITS = 256
NUM_SAMPLES_RAY = 4

# --- Phase 1: Train Depth Oracle ---
oracle_net = MLP(NUM_HIDDEN_LAYERS, HIDDEN_UNITS)

for iteration in range(NUM_ITERATIONS):
    rays, positions = sample_rays(BATCH_SIZE)
    oracle_preds = [max(0.0, min(1.0, sum(w * x for w, x in zip(oracle_net.weights[0], pos)) + oracle_net.biases[0][0])) for pos in positions]
    ground_truth_depth_bins = [random.random() for _ in range(BATCH_SIZE)]
    loss_oracle = sum(bce_loss(p, t) for p, t in zip(oracle_preds, ground_truth_depth_bins)) / BATCH_SIZE
    
    grads = [0.0]
    adam_step(oracle_net.weights[0], grads, LEARNING_RATE, iteration)

# --- Phase 2: Train Shading Network ---
shading_net = MLP(NUM_HIDDEN_LAYERS, HIDDEN_UNITS)

for iteration in range(NUM_ITERATIONS):
    rays, positions = sample_rays(BATCH_SIZE)
    depth_dist = [max(0.0, min(1.0, sum(w * x for w, x in zip(oracle_net.weights[0], pos)) + oracle_net.biases[0][0])) for pos in positions]
    warped_positions, features = warp_and_generate_features(positions, rays)
    shading_preds = [shading_net.forward(wp) for wp in warped_positions]
    ground_truth_rgb = [[random.random() for _ in range(3)] for _ in range(BATCH_SIZE)]
    
    loss_mse = sum(mse_loss(sp[:3], gt) for sp, gt in zip(shading_preds, ground_truth_rgb)) / BATCH_SIZE
    
    accumulated_opacities = [sum(sp[3] for sp in shading_preds)]
    loss_opacity = sum(max(0.0, (acc - 1.0) ** 2) for acc in accumulated_opacities) / BATCH_SIZE
    
    loss = ALPHA * loss_mse + BETA * loss_opacity
    adam_step(shading_net.weights[0], [0.0], LEARNING_RATE, iteration)
```

## Subtopics

### DONeRF Oracle Input Construction
The depth oracle network requires specific input construction to effectively guide the shading network [S1]. 3D positions along the ray are provided as additional inputs to the first network layer [S1]. These positions are placed at the centers of the discretized depth ranges [S1]. A logarithmic transformation is applied to the classification steps to align with the shading network's logarithmic sampling strategy [S1]. Positional or Fourier encoding is not used for depth oracle inputs, as experiments showed it did not improve results [S1]. The number of 3D input positions ($I$) is tested between 1 and 128, with the final configuration using $I=128$ [S1]. Providing multiple samples along the ray as input adds approximately 1 dB to PSNR [S1]. Using a classification network instead of predicting a single depth value adds 0.6 dB to PSNR [S1].

## Operating characteristics
*   **Performance:** On an NVIDIA RTX 3090 at 800x800 resolution with 2 samples per ray, total inference time is 51.3 ms (~20 fps) [S1]. Breakdown: Ray unification/feature gen (0.21 ms), Oracle network (12.64 ms), Space warping/second feature gen (3.46 ms), Shading network (34.9 ms), Final color gen (0.09 ms) [S1].
*   **Scaling:** Shading network cost scales nearly linearly with samples: 34.9 ms (2 samples), 65.4 ms (4 samples), 136.4 ms (8 samples), 270.4 ms (16 samples) [S1].
*   **Computational Cost:** Requires approximately 4.35 MFLOP per pixel [S1].
*   **Quality vs. NeRF:** Achieves equal or better quality with 2–4 samples per ray compared to NeRF's 256 [S1]. At 16 samples, achieves up to 7 dB higher PSNR and is 15–78x faster than NeRF [S1].
*   **Quality vs. NSVF:** NSVF-medium/large match DONeRF-2/4 quality but are 43x slower [S1]. NSVF requires 2x more memory for competitive quality and 6x longer training [S1].
*   **Quality vs. LLFF:** LLFF uses >1000x more memory and achieves lower quality [S1].
*   **Quality vs. NeX:** NeX is 45x faster but requires 25x more memory and is 3 dB lower quality than DONeRF-2 [S1]. NeX-MLP is better quality but DONeRF is 10x faster and 25x more compact [S1].
*   **Memory:** Does not require additional memory for explicit caching or acceleration structures [S1]. Stays within compact MLP storage requirements without baking data structures [S1].

## Background
DONeRF was introduced in the paper "DONeRF: Towards Real‐Time Rendering of Compact Neural Radiance Fields using Depth Oracle Networks" [S1]. The method addresses the computational bottleneck of standard NeRF by using a depth oracle to concentrate samples on surfaces [S1]. It builds on differentiable volume rendering but introduces a specialized two-stage network training configuration and opacity regularization to enable real-time performance on consumer hardware [S1].

## Sources
[S1] DONeRF: Towards Real‐Time Rendering of Compact Neural Radiance Fields using Depth Oracle Networks, https://doi.org/10.1111/cgf.14340, fetched 2026-10-26
