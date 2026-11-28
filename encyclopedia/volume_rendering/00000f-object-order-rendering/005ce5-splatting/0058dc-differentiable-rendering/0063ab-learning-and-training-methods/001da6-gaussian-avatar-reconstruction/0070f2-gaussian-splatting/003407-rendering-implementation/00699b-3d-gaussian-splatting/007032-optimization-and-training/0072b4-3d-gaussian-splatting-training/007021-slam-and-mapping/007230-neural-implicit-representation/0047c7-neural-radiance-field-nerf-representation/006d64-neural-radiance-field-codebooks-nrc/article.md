# Neural Radiance Field Codebooks (NRC)

## Overview
Neural Radiance Field Codebooks (NRC) is a scalable method for learning object-centric representations through novel view reconstruction, enabling the discovery of reoccurring visual and geometric patterns across scenes [S1, unofficial]. It differs from standard NeRF by mapping continuous spatial features to a discrete, finite codebook of object categories, which allows for constant rendering compute regardless of object count and supports an unbounded number of objects per scene [S1, unofficial].

## How it works
**Inputs and Outputs:**
*   **Inputs:** Input image $I_h \in \mathbb{R}^{3 \times H \times W}$, novel view image $I'_h \in \mathbb{R}^{3 \times H \times W}$, relative camera pose between input and novel view.
*   **Outputs:** Rendered novel view image, object codes $L$, encoder parameters $\theta$, variation module parameters.

**Data Structures:**
*   **Codebook $L$:** A set of $k$ categorical object codes $l_i \in \mathbb{R}^d$. The size $k$ is dynamic.
*   **Spatial Feature Map $f_h$:** Output of encoder $F_\theta$, shape $\mathbb{R}^{d \times H/k \times W/k}$.
*   **Variation Module:** A convolutional network (or MLP as per Figure 2 caption context, though text says convolutional) that transforms categorical codes to fine-grained instance codes.

**Parameters:**
*   $k$: Codebook size (number of codes). Depends on scene complexity; dynamically added [S1, unofficial].
*   $\lambda$: Scaling hyperparameter for adding new codes [S1, unofficial].
*   $\theta$: Encoder parameters [S1, unofficial].

**Steps:**
1.  **Encoding:** Process input image $I_h$ through convolutional network $F_\theta$ to obtain spatial feature map $f_h \in \mathbb{R}^{d \times H/k \times W/k}$ [S1, unofficial].
2.  **Projection:** For each point $(x, y, z)$ in world coordinates of the novel view, project to camera coordinates $(x, y)$ in the input frame using the relative camera pose [S1, unofficial].
3.  **Feature Selection:** Select the spatial feature $f_{h_{x,y}} \in \mathbb{R}^d$ from the patch in $f_h$ that contains the projected coordinates $(x, y)$ [S1, unofficial].
4.  **Code Assignment (Inference):** Assign a code $l^*$ from the shared codebook $L$ to the feature $f_{h_{x,y}}$ using arg max 1-nearest-neighbors with Straight-Through Estimator (STE) [S1, unofficial].
5.  **Code Assignment (Backward Pass):** Use a softmax relaxation of nearest-neighbors during the backward pass in conjunction with STE to enable learning of the codebook elements [S1, unofficial].
6.  **Variation Encoding:** Pass the assigned categorical code $l^*$ and the original feature vector $f_{h_{x,y}}$ to the Variation Module to transform them into fine-grained instance codes [S1, unofficial].
7.  **Rendering:** Condition a volumetric renderer on the instance code, view direction, and positional encodings to render the scene region from the novel view [S1, unofficial].
8.  **Loss Calculation:** Compare the rendered image to the ground truth novel view using L2 pixel loss [S1, unofficial].
9.  **Optimization:** Learn the categorical codes, assignment mechanism, and volumetric renderer jointly in an end-to-end fashion [S1, unofficial].
10. **Dynamic Codebook Expansion:** Add new codes to $L$ using step functions with STE to handle scene complexity [S1, unofficial].

**Formulas:**
*   **Inference Assignment (STE):**
    $$l^*(x, y) \leftarrow \arg \max_{l_i; i \in [k]} \frac{e^{-\|l_i - f_{h_{x,y}}\|^2}}{\sum_{j=1}^k e^{-\|l_j - f_{h_{x,y}}\|^2}} \quad \text{(STE)} \quad [S1, \text{unofficial}, unofficial]$$
    *Note: The text labels this as arg max but provides a softmax-like numerator/denominator structure in Eq 1, applying STE.*

*   **Backward Pass Assignment (Softmax Relaxation):**
    $$l_{back}(x, y) \leftarrow \frac{e^{-\|l_i - f_{h_{x,y}}\|^2}}{\sum_{j=1}^k e^{-\|l_j - f_{h_{x,y}}\|^2}} \quad [S1, \text{unofficial}, unofficial]$$
    *Note: The text uses $l_i$ in the numerator but implies the specific assigned code's contribution or the full softmax distribution for gradient flow.*

*   **Dynamic Code Addition:**
    $$l_i \leftarrow T\left(\sigma(s - i / \lambda)\right) \cdot l_i; \quad T(a, t) := \begin{cases} 1, & a > t \\ 0, & a \le t \end{cases} \quad [S1, \text{unofficial}, unofficial]$$
    *Where $T$ is a binarization function in the forward pass and lets gradients pass through using STE in the back pass. $\sigma(\cdot)$ is the sigmoid function, $\lambda$ is a scaling hyperparameter.*

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import torch
import torch.nn as nn
import torch.nn.functional as F

class NRC:
    def __init__(self, encoder, variation_module, renderer, codebook_size=10, lambda_param=1.0):
        self.encoder = encoder
        self.variation_module = variation_module
        self.renderer = renderer
        self.codebook_size = codebook_size
        self.lambda_param = lambda_param
        
        # Initialize codebook
        self.codes = nn.Parameter(torch.randn(codebook_size, 64)) # d=64 assumed
        
    def encode(self, image):
        # F_theta: Convolutional network
        return self.encoder(image)
        
    def project_and_select(self, feature_map, novel_view_points, camera_pose):
        # novel_view_points: (N, 3) world coordinates
        # camera_pose: relative pose matrix
        # Returns: (N, d) features from input frame
        cam_coords = transform_to_camera(novel_view_points, camera_pose)
        # Select feature from patch containing (x,y)
        # Assuming feature_map is (B, d, H/k, W/k)
        features = sample_features(feature_map, cam_coords)
        return features
        
    def assign_codes_inference(self, features, codes):
        # Eq 1 with STE
        # Compute distances
        dists = torch.cdist(features, codes) # (N, k)
        # Softmax relaxation for argmax selection via STE
        # The formula in source is softmax-like but used for argmax
        probs = torch.softmax(-dists, dim=1)
        # Argmax to get indices
        indices = torch.argmax(probs, dim=1)
        # Straight-Through Estimator: use indices in forward, but allow gradients through probs
        selected_codes = codes[indices]
        return selected_codes, indices
        
    def assign_codes_backward(self, features, codes):
        # Eq 2: Softmax relaxation for backward pass
        dists = torch.cdist(features, codes)
        probs = torch.softmax(-dists, dim=1)
        # Return the soft assignment for gradient flow
        return probs
        
    def add_code(self, step, s):
        # Eq 3: Dynamic code addition
        # T(a, t) is binarization in forward, STE in backward
        # sigma(s - i / lambda)
        # This is applied per code index i
        # Assuming s is a scalar score for adding a new code
        # This logic is complex to implement purely in PyTorch without custom autograd
        # Simplified: If step > threshold, add a new code
        if step > s / self.lambda_param:
            new_code = nn.Parameter(torch.randn(1, 64))
            self.codes = torch.cat([self.codes, new_code], dim=0)
        return self.codes
        
    def render(self, instance_codes, view_dirs, positions):
        # Condition renderer on instance code, view direction, positional encodings
        # Instance codes are fine-grained codes from variation module
        # Positional encodings for positions
        return self.renderer(instance_codes, view_dirs, positions)
        
    def train_step(self, input_image, novel_view_image, camera_pose, novel_view_points):
        # 1. Encode
        f_h = self.encode(input_image)
        
        # 2. Project and Select
        features = self.project_and_select(f_h, novel_view_points, camera_pose)
        
        # 3. Assign Codes (Inference for forward pass)
        selected_codes, indices = self.assign_codes_inference(features, self.codes)
        
        # 4. Variation Module
        # Pass categorical codes and feature vectors to variation module
        instance_codes = self.variation_module(selected_codes, features)
        
        # 5. Render
        # Assume view_dirs and positions are derived from novel_view_points
        rendered_image = self.render(instance_codes, view_dirs, novel_view_points)
        
        # 6. Loss
        loss = F.mse_loss(rendered_image, novel_view_image)
        
        # 7. Backward pass uses softmax relaxation
        # The gradients flow through the softmax relaxation defined in assign_codes_backward
        # This is handled by the STE mechanism in PyTorch if implemented correctly
        # For simplicity, we assume the forward pass uses STE and backward uses softmax
        # In practice, this requires a custom autograd function
        
        loss.backward()
        
        return loss
```

**Edge Cases and Pitfalls:**
*   **Non-differentiable Assignment:** The nearest-neighbor assignment used during inference is non-differentiable, preventing gradient propagation to the encoder and codebook [S1, unofficial]. This is mitigated by using STE and softmax relaxation during the backward pass [S1, unofficial].
*   **Dynamic Codebook:** Learning when to add new codes is non-trivial because the number and selection of codes is discrete and non-differentiable [S1, unofficial]. This is addressed using step functions with STE [S1, unofficial].

## Operating characteristics
*   **Time Complexity:** Constant rendering compute, whereas other methods' computational cost grows linearly with the number of objects [S1, unofficial].
*   **Memory Complexity:** Not given in the sources.
*   **Strengths:** Enables constant rendering compute; allows an unbounded number of objects per scene; discovers reoccurring visual and geometric patterns across scenes; transfers well to object navigation and unsupervised segmentation [S1, unofficial].
*   **Limitations:** Dynamic codebook addition is complex due to discrete nature; requires joint optimization of encoder, codebook, and renderer [S1, unofficial].
*   **When to use:** When object-centric representation is needed, especially for scenes with many objects or varying complexity, and when transferability to downstream tasks like navigation or segmentation is desired [S1, unofficial].
*   **When not to use:** When scene complexity is low and constant compute is not a priority, or when a fixed number of objects is sufficient and known a priori [S1, unofficial].

## Background
NRC was introduced in 2023 by authors associated with ICLR 2023 [S1, unofficial]. It builds on Neural Radiance Fields (NeRF) [S1, unofficial]. It addresses limitations of current 3D object-centric methods, such as fixed hyper-parameters for object count and scaling computational cost with object number [S1, unofficial]. It is evaluated on unsupervised segmentation, object navigation, and depth ordering tasks on synthetic (THOR/ProcTHOR) and real (NYU Depth) scenes [S1, unofficial].

## Sources
[S1] Neural Radiance Field Codebooks, https://doi.org/10.48550/arXiv.2301.04101, fetched 2026-10-29 (unofficial)
