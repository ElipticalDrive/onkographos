# Hierarchical Semantic Guidance

## Overview
Hierarchical Semantic Guidance (HSG) is a supervision method for radiance-field optimization that enforces semantic consistency across views by aligning coarse-to-fine semantic content with coarse-to-fine scene representations [S1, unofficial]. It solves the problem where standard CLIP-based semantic loss fails during early training because rendered images are blurred (low-frequency) while ground-truth images are high-resolution, causing a sharp drop in cosine similarity [S1, unofficial]. Unlike direct depth supervision or standard photometric loss, HSG dynamically adjusts the spatial resolution of the semantic comparison to match the learning stage of the neural field [S1, unofficial].

## How it works
**Inputs and Outputs:** The method takes a randomly selected known camera pose, the current state of the radiance field model, and the ground-truth image $C_i$ corresponding to that pose as inputs [S1, unofficial]. It outputs a scalar semantic loss value used to update the radiance field parameters [S1, unofficial].

**Data Structures:** The image is represented as $C \in \mathbb{R}^{H \times W \times 3}$ where $H$ and $W$ are height and width [S1, unofficial]. The sampling grid is defined as $P(s_i) = \{(u, v) \mid u \in [0, H, \text{stride} = s_i], v \in [0, W, \text{stride} = s_i]\}$ [S1, unofficial]. The CLIP encoder outputs a feature vector $\phi \in \mathbb{R}^D$ [S1, unofficial].

**Parameters:** $s_{max}$ is the maximum sampling stride, typically set to $0.1 \cdot \min(H, W)$ [S1, unofficial]. $N_{hsg}$ is the number of iterations until full pixels are sampled, typically 50% of total training iterations [S1, unofficial]. $i$ is the current training iteration index.

**Steps:**
1. Randomly select a known camera pose from the training set [S1, unofficial].
2. Render an image $\hat{C}_{sem}$ from the selected pose using the current radiance field model [S1, unofficial].
3. Compute the current sampling stride $s_i$ using the iteration index $i$:
   $$ s_i = \max\left(\left\lceil s_{max} \cdot \frac{1 + \cos((i/N_{hsg})\pi)}{2} \right\rceil, 1\right) $$
   where $\lceil \cdot \rceil$ denotes the ceiling operation [S1, unofficial].
4. Define the pixel sampling grid $P(s_i)$ with spacing $s_i$ [S1, unofficial].
5. Encode the rendered image $\hat{C}_{sem}$ and the ground-truth image $C_i$ using the CLIP image encoder $f_{clip}$ to obtain feature vectors $\phi_{sem} = f_{clip}(\hat{C}_{sem})$ and $\phi_i = f_{clip}(C_i)$ [S1, unofficial]. Although the CLIP encoder processes the full images, the coarse-to-fine alignment is achieved by progressively reducing $s_i$, which matches the low-frequency nature of early-stage renders to the high-frequency ground truth without requiring explicit image resizing for the CLIP input [S1, unofficial].
6. Compute the semantic loss as $1 - \text{cosine\_similarity}(\phi_{sem}, \phi_i)$ [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import math
import torch

def hierarchical_semantic_guidance_loss(render_fn, clip_encoder, pose, gt_image, iteration, n_hsg, s_max, h, w):
    """
    Computes the Hierarchical Semantic Guidance loss.
    
    Args:
        render_fn: Function that performs standard volume rendering from a camera pose [G].
        clip_encoder: Object with .preprocess() and .encode() methods for CLIP [G].
        pose: Camera pose for rendering.
        gt_image: Ground-truth image tensor of shape (H, W, 3).
        iteration: Current training iteration index (0-based).
        n_hsg: N_hsg, iterations until full sampling.
        s_max: s_max, maximum sampling stride.
        h: H, image height.
        w: W, image width.
        
    Returns:
        loss: Scalar tensor representing the semantic guidance loss.
    """
    # [G] Standard volume rendering along rays from the camera pose
    rendered_image = render_fn(pose)
    
    # Compute current stride s_i
    ratio = iteration / n_hsg
    cosine_val = math.cos(ratio * math.pi)
    stride_float = s_max * (1 + cosine_val) / 2
    s_i = max(math.ceil(stride_float), 1)
    
    # [G] Standard CLIP preprocessing: resize to 224x224, normalize to [-1, 1]
    rendered_clip = clip_encoder.preprocess(rendered_image)
    gt_clip = clip_encoder.preprocess(gt_image)
    
    # Encode features
    phi_sem = clip_encoder.encode(rendered_clip)
    phi_i = clip_encoder.encode(gt_clip)
    
    # Compute cosine similarity loss
    similarity = torch.cosine_similarity(phi_sem, phi_i, dim=0)
    loss = 1.0 - similarity
    return loss
```

**Edge Cases and Pitfalls:** CLIP semantic consistency is limited for multi-resolution images; cosine similarity decreases sharply with down-sampling rate, especially in complex real-world scenes [S1, unofficial]. The stride $s_i$ is clamped to a minimum of 1 to ensure full pixel sampling eventually [S1, unofficial]. If the rendered image is completely black or white, CLIP features may be uninformative [G].

## Subtopics

### Hierarchical Geometric Guidance (HGG)
Supervises geometry using local-to-global sampling to mitigate depth bias [S1, unofficial]. [Link](1414-hierarchical-grid-optimization.md)

### Hierarchical Photometric Guidance (HPG)
Supervises appearance using coarse and fine models [S1, unofficial]. [Link](44-radiance-field-optimization.md)

## Operating characteristics
Time complexity is not given in the sources. Memory complexity is not given in the sources. Performance improves novel view synthesis quality for sparse view inputs by providing robust semantic constraints [S1, unofficial]. Strengths include handling the mismatch between low-frequency early-stage NeRF renders and high-resolution ground truth [S1, unofficial]. Limitations include CLIP semantic consistency being limited for multi-resolution images, with potential degradation in complex scenes if coarse-to-fine alignment is not perfectly matched [S1, unofficial]. When to use and when not to use are not given in the sources.

## Background
Hierarchical Semantic Guidance was proposed in the HG3-NeRF paper for sparse view inputs [S1, unofficial]. It builds on the observation that NeRF learns low-frequency information first, resulting in blurred early renders that do not match high-resolution ground truth in feature space [S1, unofficial].

## Sources
[S1] HG3-NeRF: Hierarchical Geometric, Semantic, and Photometric Guided Neural Radiance Fields for Sparse View Inputs, https://doi.org/10.48550/arXiv.2401.11711, fetched 2026-10-29 (unofficial)
