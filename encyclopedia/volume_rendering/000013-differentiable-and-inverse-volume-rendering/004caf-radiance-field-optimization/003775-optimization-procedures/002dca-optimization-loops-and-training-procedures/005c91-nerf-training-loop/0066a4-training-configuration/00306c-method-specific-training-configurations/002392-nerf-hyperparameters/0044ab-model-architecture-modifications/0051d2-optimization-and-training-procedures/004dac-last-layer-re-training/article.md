# Last Layer Re-training

Also called: Instant NeRF Editing, Last Layer Fine-tuning

## Overview
Last Layer Re-training is a technique to instantly propagate a color edit from a single view to the entire Neural Radiance Field (NeRF) by selectively retraining only the last layer of the color Multi-Layer Perceptron (MLP) [S1]. It solves the problem of efficient NeRF editing by reducing the number of parameters requiring modification compared to full retraining, while preserving existing view-dependent information [S1]. It sets itself apart from full MLP retraining by avoiding the loss of useful pre-trained information and from methods like ICE-NeRF by preventing color bleeding at object boundaries [S1].

## How it works
**Inputs and Outputs:**
*   **Input:** A pre-trained NeRF model (specifically its color MLP weights $\phi_c$ and density MLP weights $\phi_\sigma$) and a single edited view image $I_{edit} \in \mathbb{R}^{W \times H \times 3}$ [S1].
*   **Output:** Updated NeRF parameters where the last layer of the color MLP is replaced by $\phi'_{last}$, producing a new rendered color $c'_{x,\theta}$ [S1].

**Data Structures:**
The color MLP takes as input the concatenation of multiresolution hash grid features $f_x$ and a view direction encoding $\gamma(\theta)$ [S1].
The feature encoding $f_x$ is a concatenation of features from each resolution level $L$:
$$ f_x = \{f_{x,0} \oplus f_{x,1} \oplus \dots \oplus f_{x,L}\} $$
where $\oplus$ denotes concatenation [S1].
The last layer maps the penultimate layer output $h_{x,\theta}$ to the final color [S1].

**Numbered Steps:**
1.  **Extract Penultimate Output:** For each ray sampled from the edited view, compute the hidden vector $h_{x,\theta}$ from the penultimate layer of the pre-trained color MLP. This vector depends on the position $x$ and view direction $\theta$ [S1].
2.  **Initialize Last Layer:** Initialize a new set of parameters $\phi'_{last}$ for the last layer of the color MLP. The rest of the network (density MLP and earlier layers of the color MLP) remains frozen [S1].
3.  **Compute Loss:** Render the color $C'_{ij}$ for each pixel $(i,j)$ in the edited view using the volume rendering equation with the updated last layer. Compute the photometric loss against the ground truth edited image $C_{ij}$ [S1]:
    $$ L = \sum_{i,j} \| C_{ij} - C'_{ij} \|^2 $$
    where $C_{ij}$ is the original RGB value from the training image (the edited view) and $C'_{ij}$ is the volume rendered RGB value [S1].
4.  **Optimize:** Minimize $L$ with respect to $\phi'_{last}$ using the Adam optimizer [S1].
5.  **Update Model:** Replace the original last layer weights $\phi_c$ with the optimized $\phi'_{last}$ [S1].

**Formulas:**
The naive approach would retrain the full color MLP:
$$ MLP_c(h_x, \gamma(\theta), \phi'_c) = c'_{x,\theta} $$
However, Last Layer Re-training uses:
$$ MLP_{last}(h_{x,\theta}, \phi'_{last}) = c'_{x,\theta} $$
where $h_{x,\theta}$ is the output of the penultimate layer [S1].

**Parameters:**
*   **Learning Rate:** 0.01 [S1].
*   **Optimizer:** Adam [S1].
*   **Iterations:** 200 iterations per scene [S1].

**Edge Cases and Pitfalls:**
*   **Color Bleeding:** Since the information encoded in the color MLP is global, modifying only the last layer may lead to undesired edits in regions not targeted by the user edit [S1]. This is addressed by IReNe using a soft 3D segmentation step to restrict edits to target regions [S1].
*   **View-Dependent Effects:** Editing only the last layer must preserve view-dependent effects (e.g., specularities) present in the pre-trained NeRF. The method relies on the view-dependent information already present in the pre-trained model's earlier layers [S1].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def last_layer_retraining(pretrained_nerf, edited_image, num_iterations=200, learning_rate=0.01):
    """
    Retrain only the last layer of the color MLP to match the edited image.
    
    Args:
        pretrained_nerf: Pre-trained NeRF model with frozen density MLP and color MLP layers except last.
        edited_image: Tensor of shape (H, W, 3) representing the single edited view.
        num_iterations: Number of optimization steps.
        learning_rate: Learning rate for Adam optimizer.
        
    Returns:
        updated_nerf: NeRF model with updated last layer weights.
    """
    # Extract the last layer of the color MLP
    color_mlp = pretrained_nerf.color_mlp
    last_layer = color_mlp[-1]
    
    # Freeze all parameters except the last layer
    for param in color_mlp.parameters():
        param.requires_grad = False
    for param in last_layer.parameters():
        param.requires_grad = True
        
    # Initialize optimizer for the last layer only
    optimizer = torch.optim.Adam(last_layer.parameters(), lr=learning_rate)
    
    # Render rays for the edited image to get ground truth colors
    # Assume get_rendered_colors returns tensor of shape (H, W, 3)
    # using the current last layer weights
    def render_colors():
        # Sample rays from camera poses corresponding to edited_image
        # Compute penultimate layer output h_x_theta for each sample
        # Apply last layer to get c'_x_theta
        # Volume render to get C'_ij
        return pretrained_nerf.render(edited_image.camera_poses)
        
    for step in range(num_iterations):
        optimizer.zero_grad()
        
        # Forward pass
        C_prime = render_colors()
        C_gt = edited_image # Ground truth edited image
        
        # Compute loss
        loss = torch.mean((C_gt - C_prime) ** 2)
        
        # Backward pass
        loss.backward()
        optimizer.step()
        
    # Update the pretrained_nerf with the new last layer weights
    updated_nerf = pretrained_nerf
    updated_nerf.color_mlp[-1] = last_layer
    
    return updated_nerf
```

## Subtopics

### [IReNe](13549-irene.md)
IReNe is a method for instant 360-degree recoloring of pre-trained Neural Radiance Fields (NeRFs) using a single user-edited image. It addresses the limitations of prior methods by preventing color bleeding at object boundaries and ensuring multi-view consistency for view-dependent effects, such as specularities. Unlike ICE-NeRF, which suffers from color bleeding, or Palette-NeRF, which requires hours to train, IReNe achieves high-quality edits in under 5 seconds by selectively fine-tuning only the diffuse neurons in the last layer of the color MLP.

## Operating characteristics
*   **Time Complexity:** Significantly faster than full MLP retraining because only the last layer parameters are optimized [S1].
*   **Memory Complexity:** Lower memory footprint during training compared to full retraining as gradients are only computed for the last layer [S1].
*   **Measured Performance:** Achieves high-quality edits in under 5 seconds per scene [S1].
*   **Strengths:** Rapid edits, preserves view-dependent information, prevents color bleeding (when combined with segmentation) [S1].
*   **Limitations:** May still suffer from global color bleeding if not combined with segmentation techniques [S1].
*   **When to use:** When instant editing of a pre-trained NeRF is required, and view-dependent effects must be preserved [S1].
*   **When not to use:** When the edit requires changes to the underlying geometry or density structure, or when the edit is complex and requires global feature retraining [S1].

## Background
Last Layer Re-training was introduced in the IReNe paper (Mazzucchelli et al., CVPR 2024) as a method for instant recoloring of Neural Radiance Fields [S1]. It builds on the Instant-NGP implementation [S1]. The method was developed to address the challenges of near real-time NeRF editing, specifically the trade-off between edit speed and quality [S1].

## Sources
[S1] IReNe: Instant Recoloring of Neural Radiance Fields, https://openaccess.thecvf.com/content/CVPR2024/papers/Mazzucchelli_IReNe_Instant_Recoloring_of_Neural_Radiance_Fields_CVPR_2024_paper.pdf, fetched 2026-10-29
