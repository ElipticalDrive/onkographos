# NARF Training

Also called: NARF Optimization

## Overview
NARF (Neural Articulated Radiance Field) training optimizes a neural network to represent the radiance field of articulated 3D objects by minimizing photometric and mask errors between rendered images and ground truth inputs [S1]. It distinguishes itself from standard NeRF training by incorporating known kinematic pose configurations $\{T_0, \zeta, \theta\}$ into the rendering pipeline, allowing the model to disentangle geometry from articulation [S1]. Among its variants, Disentangled NARF (NARFD) is preferred for its balance of computational efficiency and ability to generate segmentation masks, outperforming Part-Wise and Holistic variants in novel pose/view settings [S1].

## How it works
**Inputs:**
*   Set of training images with known camera poses [S1].
*   Kinematic 3D pose configuration for the articulated object: global rigid transformation $T_0$, bone parameters $\zeta$, and joint angles $\theta$ [S1].
*   Ground truth pixel colors $\mathbf{C}(\mathbf{r})$ and foreground masks $\mathbf{M}(\mathbf{r})$ [S1].

**Outputs:**
*   Rendered color $\hat{\mathbf{C}}(\mathbf{r})$ and mask $\hat{\mathbf{M}}(\mathbf{r})$ for each ray $\mathbf{r}$ [S1].
*   Optimized network parameters $\Theta$ [S1].

**Data Structures:**
*   **NARF Model:** A single Multi-Layer Perceptron (MLP) mapping spatial coordinates and viewing directions to density $\sigma$, feature vector $\mathbf{h}$, and color $C$ [S1].
*   **Selector $S$ (NARFD only):** A set of $P$ lightweight sub-networks, one for each object part [S1].
    *   Each sub-network $O_\Gamma^i$ is a two-layer MLP with 10 hidden nodes [S1].
    *   Input: Local 3D position $\mathbf{x}^i$ and bone parameter $\zeta$ [S1].
    *   Output: Logit $o^i$, normalized to probability $p^i$ via softmax [S1].

**Numbered Steps:**
1.  **Ray Sampling:** Randomly sample a batch of camera rays $\mathcal{R}$ from all pixels [S1].
2.  **Hierarchical Sampling:** For each ray, query $N$ samples using the hierarchical volume sampling strategy from original NeRF [S1].
3.  **Coordinate Transformation:** For each sample at global position $\mathbf{x}$ and direction $\mathbf{d}$, compute local coordinates for each part $i$ using the known kinematic pose $\{T_0, \zeta, \theta\}$:
    $$ \mathbf{x}^i = R^{i-1}(\mathbf{x} - \mathbf{t}^i), \quad \mathbf{d}^i = R^{i-1}\mathbf{d} $$
    where $R^{i-1}$ and $\mathbf{t}^i$ are the rotation and translation of part $i$ [S1].
4.  **Positional Encoding:** Apply positional encoding $\gamma(\cdot)$ to inputs. Dimensions are set to 10 for 3D location $\mathbf{x}$ and 4 for other parameters (directions, bone parameters) [S1].
5.  **Selector Probability (NARFD only):** For each part $i$, compute the probability $p^i$ that $\mathbf{x}$ belongs to part $i$:
    $$ O_\Gamma^i : (\gamma(\mathbf{x}^i), \gamma(\boldsymbol{\zeta})) \rightarrow (o^i), \quad p^i = \frac{\exp(o^i)}{\sum_{k=1}^P \exp(o^k)} $$
    [S1].
6.  **Network Forward Pass:**
    *   Construct the input vector by concatenating the masked local features. For each part $i$, mask the encoded local position and direction with the selector probability $p^i$:
        $$ \text{Input}_\sigma = \text{Cat}(\{\gamma(\mathbf{x}^i) * p^i \mid i \in [1, P]\}, \gamma(\zeta)) $$
        $$ \text{Input}_c = \text{Cat}(\mathbf{h}, \{(\gamma(\mathbf{d}^i) * p^i, \gamma(\boldsymbol{\xi}^i) * p^i) \mid i \in [1, P]\}) $$
        where $\mathbf{h}$ is the intermediate feature vector from the density network [S1].
    *   Query the NARF model to obtain density $\sigma$, feature $\mathbf{h}$, and color $C$ [S1].
7.  **Volume Rendering:** Render the color $\hat{\mathbf{C}}(\mathbf{r})$ and mask $\hat{\mathbf{M}}(\mathbf{r})$ for the ray using standard volume rendering equations (Eq. 5 and Eq. 16 in [S1]) [S1].
8.  **Loss Computation:** Compute the total squared error loss:
    $$ \mathcal{L} = \sum_{\mathbf{r} \in \mathcal{R}} \|\hat{\mathbf{C}}(\mathbf{r}) - \mathbf{C}(\mathbf{r})\|_2^2 + \|\hat{\mathbf{M}}(\mathbf{r}) - \mathbf{M}(\mathbf{r})\|_2^2 $$
    where $\mathbf{C}$ and $\mathbf{M}$ are ground truth color and foreground mask [S1].
9.  **Optimization:** Update network parameters using gradient descent [S1].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import math

def pos_enc(x, dim):
    # Standard positional encoding expanding x to dim dimensions [G]
    return x  # Conceptual representation; actual implementation uses sin/cos frequencies [G]

def concatenate(vectors):
    return [val for vec in vectors for val in vec]

def transform_to_local_part(x_global, part_idx, poses):
    R = poses.rotation(part_idx)
    t = poses.translation(part_idx)
    return [sum(R[j][k] * (x_global[k] - t[k]) for k in range(3)) for j in range(3)]

def transform_direction(d_global, part_idx, poses):
    R = poses.rotation(part_idx)
    return [sum(R[j][k] * d_global[k] for k in range(3)) for j in range(3)]

def softmax(logits):
    m = max(logits)
    exps = [math.exp(l - m) for l in logits]
    s = sum(exps)
    return [e / s for e in exps]

def volume_render(t_vals, sigma, color, mask):
    T = 1.0
    rendered_color = [0.0, 0.0, 0.0]
    rendered_mask = 0.0
    for i in range(len(t_vals)):
        t_i = t_vals[i]
        t_prev = t_vals[i-1] if i > 0 else t_vals[0] - 0.5
        dist = t_i - t_prev
        alpha = 1.0 - math.exp(-sigma[i] * dist)
        weight = T * alpha
        rendered_color = [rendered_color[j] + weight * color[i][j] for j in range(3)]
        rendered_mask += weight * mask[i]
        T *= (1.0 - alpha)
    return rendered_color, rendered_mask

def train_narf(batch_rays, poses, gt_colors, gt_masks, nerf_model, selector=None, num_samples=64):
    total_loss = 0.0
    num_parts = len(poses.parts) if hasattr(poses, 'parts') else (len(selector) if selector else 1)

    for ray, gt_color, gt_mask in zip(batch_rays, gt_colors, gt_masks):
        # 1. Hierarchical sampling (standard NeRF strategy [G])
        t_vals = [ray.origin_norm + i * 0.5 for i in range(num_samples)]
        samples = [(t, [ray.origin[j] + t * ray.direction[j] for j in range(3)]) for t in t_vals]

        # 2. Transform coordinates & compute selector probabilities
        probs = []
        if selector is not None:
            for i in range(num_parts):
                x_local = transform_to_local_part(samples[0][1], i, poses)
                logit = selector[i].forward(pos_enc(x_local, 10), pos_enc(poses.zeta, 4))
                probs.append(logit)
            probs = softmax(probs)

        sigma_list, color_list, mask_list = [], [], []
        for t, x_global in samples:
            sigma_inputs = []
            color_inputs = []
            for i in range(num_parts):
                x_local = transform_to_local_part(x_global, i, poses)
                d_local = transform_direction(ray.direction, i, poses)
                # Masked inputs per Eq 22 & 23
                sigma_inputs.append([v * probs[i] for v in pos_enc(x_local, 10)])
                color_inputs.append([v * probs[i] for v in pos_enc(d_local, 4)])

            sigma_input_vec = concatenate(sigma_inputs + [pos_enc(poses.zeta, 4)])
            sigma, h = nerf_model.forward_sigma(sigma_input_vec)

            color_input_vec = [h] + color_inputs
            color = nerf_model.forward_color(concatenate(color_input_vec))
            sigma_list.append(sigma)
            color_list.append(color)
            mask_list.append(1.0) # Standard foreground mask assumption [G]

        # 3. Volume Rendering
        rendered_color, rendered_mask = volume_render(t_vals, sigma_list, color_list, mask_list)

        # 4. Loss
        color_loss = sum((c - g)**2 for c, g in zip(rendered_color, gt_color))
        mask_loss = (rendered_mask - gt_mask)**2
        total_loss += color_loss + mask_loss

    return total_loss / len(batch_rays)
```

**Parameters:**
*   Positional encoding dimension for 3D location $\mathbf{x}$: 10 [S1].
*   Positional encoding dimension for other parameters (directions, bone parameters): 4 [S1].
*   Number of samples per ray $N$: Not given in the sources [S1].
*   Selector network architecture: Two-layer MLP with 10 hidden nodes per part [S1].

**Edge Cases and Pitfalls:**
*   **Input Concatenation:** Although irrelevant parts are masked, the input remains a concatenation of all parts. This is intentional to allow the single NeRF to distinguish different bones via the dimensions of the concatenated vector, similar to part identity encoding [S1].
*   **Selector Output:** The selector outputs probabilities for *all* parts, even if a location belongs to only one. The softmax ensures normalization [S1].

## Subtopics

### Disentangled NARF
An approach that shares the merits of both Part-Wise NARF (NARFP) and Holistic NARF (NARFH) while avoiding their weaknesses by introducing a selector $S$ to identify which object part a global 3D location belongs to [S1]. It uses lightweight sub-networks to output part probabilities, which mask the concatenated inputs to the main NeRF [S1]. It outperforms other methods under all evaluation metrics and test data settings, exhibiting high performance under novel pose and/or novel view settings with low computational cost close to a single NeRF model [S1].

## Operating characteristics
*   **Time Complexity:** Computational cost is close to a single NeRF model, as it uses one unified network rather than separate networks per part [S1].
*   **Memory Complexity:** Proportional to the number of elements to preserve during forward propagation per ray, which is similar to a single NeRF [S1].
*   **Performance:** NARFD outperforms NARFP and NARFH under all evaluation metrics and test data settings [S1]. It shows slight performance drop on SSIM (within 4%) in novel pose/view settings but maintains high performance [S1].
*   **Strengths:** Avoids the zero density problem of NARFH; satisfies Part Dependency unlike NARFH; generates segmentation masks; low computational cost [S1].
*   **Limitations:** Requires known kinematic 3D pose configuration $\{T_0, \zeta, \theta\}$ [S1].
*   **When to use:** When reconstructing articulated 3D objects with known kinematics and requiring efficient training with segmentation capabilities [S1].
*   **When not to use:** When kinematic poses are unknown or when the object is not articulated [S1].

## Background
NARF was introduced in the paper "Neural Articulated Radiance Field" [S1]. It addresses the challenge of representing articulated 3D objects by combining neural radiance fields with kinematic models [S1]. The method was developed to overcome the limitations of Part-Wise and Holistic NARF approaches, offering a disentangled representation that is both efficient and accurate [S1].

## Sources
[S1] Neural Articulated Radiance Field, https://doi.org/10.1109/iccv48922.2021.00571, fetched 2026-10-26
