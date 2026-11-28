# GSGAN Hierarchical Generation

Also called: GSGAN

## Overview
GSGAN is a generative model that synthesizes 3D Gaussian splats hierarchically to approximate complex 3D shapes and textures, building on the primitive representation defined in [Gaussian Parameter Prediction](18303-gaussian-parameter-prediction.md) [S1]. It differs from standard 3D Gaussian Splatting by generating the Gaussian parameters via a deep neural network conditioned on pose and camera views, rather than optimizing them from static point clouds [S1]. This allows for the generation of novel 3D objects from 2D images [S1].

## How it works
**Inputs and Outputs:** The method takes an intermediate feature tensor $x_l$ at level $l$ and a camera parameter vector $\theta$ as input [S1]. It outputs a set of Gaussian parameters: opacity $\hat{\alpha}$, quaternion orientation $\hat{q}$, position $\hat{\mu}$, scale difference $\hat{s}$, and color $\hat{c}$ [S1]. The final scale is derived from $\hat{s}$ and a hierarchical constant $\Delta s$ [S1].

**Data Structures:** The Gaussian set is organized hierarchically across $L$ levels [S1]. Level $l=0$ starts with $N=256$ Gaussians [S1]. Each level upsamples by a ratio $r=4$ [S1]. Total Gaussians are 87,000 for $256 \times 256$ resolution ($L=5$) and 349,000 for $512 \times 512$ resolution ($L=6$) [S1]. Channel dimensions per level $[0, 1, 2, 3, 4, 5]$ are $[512, 512, 256, 128, 128, 128]$ [S1].

**Numbered Steps:**
1. **Initialization:** Initialize the bias of the linear layers to set the quaternion to an identity rotation matrix, the opacity to 0.1, and the scale to $e^{\frac{\sqrt{\Delta s}}{N}}$ [S1].
2. **Feature Propagation:** For each level $l$ from 0 to $L-1$:
    a. **Attention Mechanism:** Compute attention scores between Gaussians to model interactions [S1]. For $l=0$, use general self-attention [S1]. For $0 < l \le 3$, use local attention similar to EdgeConv, computing attention scores against $K$-nearest neighbors ($K=16$) [S1]. For $l > 3$, model interactions between points generated from the identical parent point in level $l-1$, rather than computing actual $K$-NN [S1].
    b. **Parameter Synthesis (toGaus Layer):** Apply 5 parallel linear layers to the intermediate feature $x_l$ to predict parameters [S1].
3. **Parameter Activation:** Apply specific activation functions to the linear outputs [S1]. Position $\hat{\mu}$ uses $\tanh$ activation [S1]. Scale difference $\hat{s}$ uses $-\text{softplus}$ activation [S1]. Color $\hat{c}$ is the linear output [S1]. Quaternion $\hat{q}$ is normalized to unit length [S1]. Opacity $\hat{\alpha}$ uses sigmoid activation (implied by 0.1 initialization) [G]. In the rendering process, an exponential activation function is applied to the scale parameters [S1].
4. **Hierarchical Scaling:** Calculate the scale reduction constant $\Delta s$ based on resolution and hierarchy depth [S1]. The final scale is derived from the predicted $\hat{s}$ and $\Delta s$ [S1].

**Formulas:**
The output of the `toGaus` layer at level $l$ is defined as [S1]:
$$
p_l = \text{Lin}_p(x_l)
$$
$$
\hat{\mu}_l = \tanh(\text{Lin}_{\hat{\mu}}(x_l))
$$
$$
\hat{s}_l = -\text{softplus}(\text{Lin}_{\hat{s}}(x_l))
$$
$$
\hat{c}_l = \text{Lin}_{\hat{c}}(x_l, \theta)
$$
where $p \in \{\hat{\alpha}, \hat{q}\}$ and $\text{Lin}_{(\cdot)}$ is a linear layer [S1]. The constant $\Delta s$ is calculated as [S1]:
$$
\Delta s = -\log\left(\sqrt{\frac{H \times W}{L \times N}}\right)
$$
where $H, W$ are image height and width, $L$ is the total number of hierarchical levels, and $N$ is the initial number of Gaussians at level $l=0$ [S1].

**Parameters:**
- Upsampling ratio $r = 4$ [S1].
- Initial Gaussians $N = 256$ [S1].
- Hierarchical levels $L = 5$ for $256 \times 256$ resolution, $L = 6$ for $512 \times 512$ resolution [S1].
- Objective function coefficients: $\lambda = 1$, $\lambda_{pose} = 1$, $\lambda_{knn} = 10$ [S1].
- $\lambda_{center} = 1$ for FFHQ dataset, $\lambda_{center} = 10$ for AFHQ-Cat dataset [S1].
- Training duration: Discriminator sees 10–15 million images [S1].
- AdaIN and adaptive layerscale are applied only to layers lower than level 3 [S1].
- Equalized learning rate is used for every layer [S1].

**Edge Cases and Pitfalls:**
- Computational cost of self-attention and KNN grows with the number of Gaussians. The method switches to parent-point interaction for $l>3$ to mitigate this [S1].
- Modulation (AdaIN) on higher levels ($l \ge 3$) does not significantly affect rendering results and is omitted to save computation [S1].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import math

def initialize_biases(num_gaussians, delta_s):
    q_bias = [1.0, 0.0, 0.0, 0.0]  # Identity quaternion [w, x, y, z]
    alpha_bias = math.log(0.1 / 0.9)
    target_scale = math.exp(math.sqrt(delta_s) / num_gaussians)
    s_bias = target_scale
    return q_bias, alpha_bias, s_bias

def compute_attention(points, level, parent_points=None):
    n = len(points)
    if level == 0:
        scores = [[sum(points[i][k] * points[j][k] for k in range(len(points[0]))) for j in range(n)] for i in range(n)]
        return scores
    elif level <= 3:
        knn_indices = []
        for i in range(n):
            dists = [sum((points[i][k] - points[j][k])**2 for k in range(len(points[0]))) for j in range(n)]
            sorted_indices = sorted(range(n), key=lambda k: dists[k])
            knn_indices.append(sorted_indices[:16])
        scores = []
        for i in range(n):
            row = [sum(points[i][k] * points[j][k] for k in range(len(points[0]))) for j in knn_indices[i]]
            scores.append(row)
        return scores
    else:
        m = len(parent_points)
        scores = [[sum(points[i][k] * parent_points[j][k] for k in range(len(points[0]))) for j in range(m)] for i in range(n)]
        return scores

def toGaus_layer(x_l, theta):
    dim = len(x_l)
    theta_dim = len(theta)
    
    p_lin = sum(x_l[k] for k in range(dim))
    mu_lin = [x_l[k] for k in range(dim)]
    s_lin = [x_l[k] for k in range(dim)]
    c_lin = [x_l[k] + theta[k % theta_dim] for k in range(dim + theta_dim)]
    q_lin = [x_l[k] for k in range(dim)]
    
    alpha_hat = 1.0 / (1.0 + math.exp(-p_lin))
    mu_hat = [math.tanh(v) for v in mu_lin]
    s_hat = [-math.log(1 + math.exp(v)) for v in s_lin]
    c_hat = c_lin
    norm_q = math.sqrt(sum(v**2 for v in q_lin))
    q_hat = [v / norm_q for v in q_lin]
    
    return alpha_hat, q_hat, mu_hat, s_hat, c_hat

def generate_gaussians(image_resolution, num_levels, initial_gaussians):
    H, W = image_resolution
    N = initial_gaussians
    L = num_levels
    
    delta_s = -math.log(math.sqrt((H * W) / (L * N)))
    
    x_l = [[1.0] * 512] * N
    theta = [0.0] * 128
    
    all_gaussians = []
    
    for l in range(L):
        parent_points = all_gaussians[-1] if l > 0 else None
        attention_scores = compute_attention(x_l, l, parent_points)
        
        x_l = [[sum(attention_scores[i][j] * x_l[j][k] for j in range(len(x_l))) for k in range(len(x_l[0]))] for i in range(len(x_l))]
        
        alpha_hat, q_hat, mu_hat, s_hat, c_hat = toGaus_layer(x_l[0], theta)
        
        level_gaussians = [alpha_hat] + q_hat + mu_hat + s_hat + c_hat
        all_gaussians.append(level_gaussians)
        
        if l < L - 1:
            x_l = x_l * 4
        
    return all_gaussians
```

## Operating characteristics
- **Time Complexity:** Self-attention at level 0 is $O(N^2)$. Local attention at levels 1-3 is $O(N \cdot K)$ where $K=16$. Parent-point interaction at levels 4+ is $O(N)$. The hierarchical structure keeps the total computational cost manageable compared to global self-attention on the full set [S1].
- **Memory Complexity:** Stores Gaussian parameters for all levels simultaneously during generation. Total Gaussians: 87K ($256^2$) or 349K ($512^2$) [S1].
- **Strengths:** Generates high-fidelity 3D structures with hierarchical detail. Efficient attention mechanisms allow scaling to higher resolutions [S1].
- **Limitations:** Requires significant training time (10-15M images for discriminator) [S1]. Performance depends on the quality of the pose conditioning [S1].
- **When to use:** When generating novel 3D objects from 2D images is required, and hierarchical detail is beneficial [S1].
- **When not to use:** When real-time rendering of a fixed set of Gaussians is the only goal without generation capability [G].

## Background
GSGAN was introduced in the paper "GSGAN: Adversarial Learning for Hierarchical Generation of 3D Gaussian Splats" [S1]. It builds upon the 3D Gaussian Splatting representation [22] and the StyleGAN2 architecture [12, 58]. The method was developed to enable the generation of 3D-aware images using Gaussian primitives, extending the capabilities of GANs to 3D space [S1].

## Sources
[S1] GSGAN: Adversarial Learning for Hierarchical Generation of 3D Gaussian Splats, https://doi.org/10.52202/079017-2171, fetched 2026-10-29
