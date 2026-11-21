# Weight Freezing by Mask

Also called: freezweightsbymask, freezweightsbymasknounsqueeze

## Overview
Weight Freezing by Mask is a training optimization technique for 3D Gaussian Splatting that selectively disables gradient updates for specific Gaussian attributes (such as rotation or opacity) by applying a binary mask to their gradient tensors during the back-propagation step [S1, unofficial]. It differs from global parameter freezing by allowing fine-grained, per-element control over which Gaussians contribute to the optimization of specific parameters, thereby stabilizing training or enforcing structural constraints without removing Gaussians from the scene [S1, unofficial].

## How it works
The method operates during the gradient computation phase, after the backward pass has calculated gradients for all learnable parameters but before the optimizer applies them. It targets specific attributes defined in a `screenlist`.

**Inputs and Outputs:**
*   **Inputs:**
    *   `gradients`: A dictionary or list of tensors containing the computed gradients for each Gaussian attribute (e.g., `_omega`, `_rotation`, `_opacity`).
    *   `masks`: A binary tensor of the same shape as the gradient tensor (or broadcastable to it), where `1` indicates the gradient should be preserved and `0` indicates it should be zeroed.
    *   `screenlist`: A list of strings specifying the attribute names to process (e.g., `['_omega']`) [S1, unofficial].
*   **Outputs:**
    *   Modified `gradients` dictionary/list where selected entries have zeroed gradients.

**Data Structures:**
*   **Gradient Tensor:** Shape $(N, D)$, where $N$ is the number of Gaussians and $D$ is the dimensionality of the attribute (e.g., $D=4$ for quaternion rotation, $D=1$ for opacity).
*   **Mask Tensor:** Shape $(N, 1)$ or $(N, D)$, containing values in $\{0, 1\}$.

**Steps:**
1.  Iterate through each attribute name `attr` in `screenlist`.
2.  Retrieve the gradient tensor $G_{attr}$ for `attr`.
3.  Retrieve the corresponding mask tensor $M_{attr}$.
4.  Apply the mask to the gradient:
    *   If the variant is `freezweightsbymask`, unsqueeze $M_{attr}$ by one dimension to match $G_{attr}$'s shape if necessary, then compute $G'_{attr} = G_{attr} \odot M_{attr}$, where $\odot$ is element-wise multiplication [S1, unofficial].
    *   If the variant is `freezweightsbymasknounsqueeze`, multiply $G_{attr}$ directly by $M_{attr}$ without unsqueezing, relying on broadcasting rules [S1, unofficial].
5.  Set the modified gradient $G'_{attr}$ back into the gradient dictionary/list.

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def apply_weight_freezing_by_mask(gradients, masks, screenlist, variant='freezweightsbymask'):
    """
    Freezes gradients for specific Gaussian attributes using a mask.
    
    Args:
        gradients: Dict[str, torch.Tensor] mapping attribute names to gradient tensors.
        masks: Dict[str, torch.Tensor] mapping attribute names to mask tensors.
        screenlist: List[str] of attribute names to process.
        variant: str, either 'freezweightsbymask' or 'freezweightsbymasknounsqueeze'.
        
    Returns:
        Dict[str, torch.Tensor] with modified gradients.
    """
    for attr in screenlist:
        if attr not in gradients or attr not in masks:
            continue
            
        grad = gradients[attr]
        mask = masks[attr]
        
        if variant == 'freezweightsbymask':
            # Unsqueeze mask to match gradient dimensions if needed
            # Assuming mask is (N, 1) and grad is (N, D)
            if mask.dim() == grad.dim() - 1:
                mask = mask.unsqueeze(-1)
            # Element-wise multiplication
            new_grad = grad * mask
        elif variant == 'freezweightsbymasknounsqueeze':
            # Direct multiplication, relying on broadcasting
            new_grad = grad * mask
        else:
            raise ValueError(f"Unknown variant: {variant}")
            
        gradients[attr] = new_grad
        
    return gradients
```

**Parameters:**
*   `screenlist`: List of attribute names (e.g., `['_omega']`, `['_rotation']`) [S1, unofficial].
*   `variant`: String determining mask handling ('freezweightsbymask' or 'freezweightsbymasknounsqueeze') [S1, unofficial].

**Edge Cases and Pitfalls:**
*   **Shape Mismatch:** If the mask shape does not broadcast correctly with the gradient tensor, the operation will fail or produce incorrect results. The `freezweightsbymask` variant explicitly handles this by unsqueezing [S1, unofficial].
*   **Precision:** Using binary masks (0/1) is standard. Using soft masks (values in [0,1]) effectively scales the gradient, which may be used for regularization but is not the primary "freezing" mechanism described [S1, unofficial].
*   **Degenerate Input:** If a mask contains all zeros, the gradient for that attribute becomes zero, effectively freezing it for that step. If all ones, no change occurs.

## Subtopics

### Parameter Freezing
A simpler variant that freezes weights for entire parameters rather than per-element. It iterates over named parameters and sets `requires_grad` to `False` if the parameter key starts with 'decoder' or is 'color_decoder' [S2, unofficial]. This disables gradient computation entirely for those parameters, rather than zeroing gradients after computation.

## Operating characteristics
*   **Time Complexity:** $O(N \cdot D)$, where $N$ is the number of Gaussians and $D$ is the attribute dimensionality. The operation is a simple element-wise multiplication, which is highly parallelizable on GPU.
*   **Memory Complexity:** $O(N \cdot D)$ for storing the mask tensor.
*   **Strengths:** Allows selective optimization, preventing unwanted updates to stable attributes (e.g., fixing rotation while optimizing position). Can stabilize training in dynamic scenes.
*   **Limitations:** Requires pre-computed or dynamically generated masks. Does not reduce memory usage during the forward pass, only affects the optimizer step.
*   **When to use:** When specific attributes need to be constrained or stabilized during training, or when implementing ablation studies on attribute contributions.

## Background
Weight Freezing by Mask is a technique used in advanced 3D Gaussian Splatting training pipelines, particularly in research involving dynamic scenes or specific structural constraints. It is documented in the SpacetimeGaussians repository [S1, unofficial] and related works. The concept of freezing parameters is common in deep learning, but the per-element masking approach is specific to the large-scale, per-Gaussian optimization of 3DGS.

## Sources
[S1] oppo-us-research/SpacetimeGaussians, https://github.com/oppo-us-research/SpacetimeGaussians, fetched 2026-10-27 (unofficial)
[S2] hku-mars/GS-SDF, https://github.com/hku-mars/GS-SDF, fetched 2026-10-27 (unofficial)
