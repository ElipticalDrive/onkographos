# Promptable Explicit Scene Representation

Also called: Semantic Prompting

## Overview
A mechanism for 3D editing (extraction, deletion, appearance modification) in 3D Gaussian Splatting by computing activation scores of 3D Gaussians based on prompts (text, points, boxes) using cosine similarity and softmax [S1]. It differs from implicit NeRF feature fields by operating on explicit Gaussian primitives, allowing direct manipulation of color and opacity parameters for specific semantic subsets [S1].

## How it works
**Inputs:**
- A set of $N$ 3D Gaussians $\{x_i\}_{i=1}^N$, where each Gaussian $x$ has a semantic feature vector $f(x)$, color $c(x)$, and opacity $\alpha(x)$.
- A prompt $\tau$ (text, point, or box).
- For text prompts: A pre-trained CLIP ViT-B/32 encoder [S1].
- A set of possible labels $\mathcal{T}$ (e.g., text labels for segmentation or pixel coordinates for point prompts) [S1].

**Outputs:**
- A "Gaussian mask" identifying Gaussians associated with the prompt.
- An edited radiance field with updated $\{c_i\}$ and $\{\alpha_i\}$ for selected Gaussians, where the count $n < N$ [S1].

**Data Structures:**
- **Gaussian Feature:** Each Gaussian $x$ stores a semantic feature vector $f(x)$. The dimensionality is determined by the feature field distillation process (e.g., matching CLIP embedding dimensions) [S1].
- **Query Vector:** For text, $q(\tau)$ is the CLIP text embedding [S1]. For points/boxes, $q(\tau)$ corresponds to the 2D pixel feature projected into 3D via distillation [S1].

**Steps:**
1. **Encode Prompt:** If $\tau$ is text, compute $q(\tau)$ using the CLIP ViT-B/32 encoder [S1]. If $\tau$ is a point or box, retrieve the corresponding 2D feature from the distilled feature field and map it to the 3D Gaussian space [S1].
2. **Compute Activation Score:** For each Gaussian $x$, calculate the cosine similarity between its semantic feature $f(x)$ and the query vector $q(\tau)$:
   $$ s = \frac {f(x) \cdot q(\tau )}{\|f(x)\| \|q(\tau )\|} $$
   where $s$ is the activation score [S1].
3. **Compute Probability:** If a set of labels $\mathcal{T}$ is provided, compute the probability $\mathbf {p}(\tau | x)$ using softmax over all scores $s_j \in \mathcal{T}$:
   $$ \mathbf {p}(\tau | x) = \text {softmax} (s) = \frac {\exp \left ( s \right )}{\sum _{s_j \in \mathcal {T}} \exp \left (s_j\right )} $$
   [S1].
4. **Generate Gaussian Mask:** Filter Gaussians based on the computed probabilities or scores. Two selection modes are supported [S1]:
   - **Soft Selection:** Threshold the score matrix to include Gaussians with scores above a certain value.
   - **Hard Selection:** Filter based on the highest score across $K$ categories (where $K$ is the number of object categories) [S1].
5. **Edit Scene:** Update the color $c(x)$ and opacity $\alpha(x)$ of the selected Gaussians. For extraction, set opacity to 0; for deletion, remove or zero out; for appearance modification, change color [S1].
6. **Render:** Render the edited radiance field using point-based $\alpha$-blending with the updated set $\{c_i\}_{i=1}^n$ and $\{\alpha_i\}_{i=1}^n$, where $n < N$ [S1].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def promptable_editing(gaussians, prompt, clip_encoder, labels=None, threshold=None):
    """
    gaussians: List of Gaussian objects, each with .feature (f(x)), .color (c), .opacity (alpha)
    prompt: Text string, or (x, y) point coordinates, or (x1, y1, x2, y2) box coordinates
    clip_encoder: Function to encode text to CLIP embedding
    labels: Optional list of label strings for softmax normalization
    threshold: Optional float for soft selection
    """
    # Step 1: Encode Prompt
    if isinstance(prompt, str):
        query_vector = clip_encoder.encode(prompt)
    else:
        # For point/box, retrieve feature from distilled 2D feature field
        # This assumes a global feature field lookup function exists
        query_vector = feature_field.lookup(prompt)
    
    selected_gaussians = []
    score_matrix = []
    
    # Step 2 & 3: Compute Scores and Probabilities
    for g in gaussians:
        # Cosine similarity
        dot_product = dot(g.feature, query_vector)
        norm_f = norm(g.feature)
        norm_q = norm(query_vector)
        
        if norm_f == 0 or norm_q == 0:
            score = 0.0
        else:
            score = dot_product / (norm_f * norm_q)
        
        score_matrix.append(score)
        
        # If labels are provided, compute softmax probability
        if labels is not None:
            # Compute scores for all labels to normalize
            all_scores = []
            for label in labels:
                q_label = clip_encoder.encode(label)
                s_label = dot(g.feature, q_label) / (norm_f * norm(q_label))
                all_scores.append(s_label)
            
            # Softmax
            exp_scores = [exp(s) for s in all_scores]
            sum_exp = sum(exp_scores)
            if sum_exp == 0:
                prob = 0.0
            else:
                # Find index of current prompt's score in all_scores to get its probability
                # Note: In practice, query_vector is derived from one of the labels or a specific prompt
                # Here we assume we want the probability of the current prompt relative to the set
                idx = labels.index(prompt) if prompt in labels else -1
                if idx != -1:
                    prob = exp_scores[idx] / sum_exp
                else:
                    prob = 0.0
        else:
            prob = score # Use raw score if no labels for normalization
            
        # Step 4: Generate Mask
        is_selected = False
        if threshold is not None:
            if prob >= threshold:
                is_selected = True
        else:
            # Hard selection: highest score across K categories
            # Assuming 'labels' defines the K categories
            if labels is not None and prob > 0: # Simplified logic for hard selection
                 # In hard selection, we typically pick the argmax category
                 # If the prompt matches the argmax category, select
                 max_prob = max(exp_scores) / sum_exp if sum_exp > 0 else 0
                 if prob == max_prob:
                     is_selected = True
                     
        if is_selected:
            selected_gaussians.append(g)
            
    # Step 5: Edit Scene
    # Example: Extraction (set opacity to 0)
    for g in gaussians:
        if g in selected_gaussians:
            g.opacity = 0.0
        else:
            # Optional: Keep others, or remove them from rendering list
            pass
            
    # Step 6: Render
    # Render using alpha blending with updated gaussians
    image = render_alpha_blend(selected_gaussians)
    return image, selected_gaussians
```

**Edge Cases and Pitfalls:**
- **Zero Norm:** If $f(x)$ or $q(\tau)$ has zero norm, the cosine similarity is undefined. Handle by setting score to 0 [G].
- **Implicit vs Explicit:** Unlike NeRF-based feature fields where the feature branch shares layers with the radiance field (potentially causing interference), this method uses explicit Gaussian features, avoiding such overlap issues [S1].
- **Distillation Dependency:** The quality of $f(x)$ depends on the teacher-student distillation process. Poor distillation leads to inaccurate semantic features [S1].

## Operating characteristics
- **Time Complexity:** Computing scores is $O(N \cdot D)$ where $N$ is the number of Gaussians and $D$ is the feature dimension. Softmax is $O(N \cdot K)$ where $K$ is the number of labels. Rendering is $O(N)$ with $\alpha$-blending [S1].
- **Memory Complexity:** Requires storing semantic features $f(x)$ for each Gaussian, increasing memory usage compared to standard 3DGS [S1].
- **Strengths:** Enables direct 3D editing (extraction, deletion, modification) using 2D prompts (points, boxes) or text. Leverages pre-trained models (CLIP, SAM) for semantic understanding [S1].
- **Limitations:** Relies on the accuracy of the distilled feature field. Text prompts require CLIP encoding. Hard selection may be less flexible than soft thresholding [S1].

## Background
Originated in "Feature 3DGS: Supercharging 3D Gaussian Splatting to Enable Distilled Feature Fields" (CVPR 2024) [S1]. It extends 2D pixel-level promptable editing (from SAM and CLIP) into the 3D realm by distilling 2D features into a 3D feature field associated with 3D Gaussians [S1].

## Sources
[S1] Feature 3DGS: Supercharging 3D Gaussian Splatting to Enable Distilled Feature Fields, https://doi.org/10.1109/cvpr52733.2024.02048, fetched 2026-10-26
