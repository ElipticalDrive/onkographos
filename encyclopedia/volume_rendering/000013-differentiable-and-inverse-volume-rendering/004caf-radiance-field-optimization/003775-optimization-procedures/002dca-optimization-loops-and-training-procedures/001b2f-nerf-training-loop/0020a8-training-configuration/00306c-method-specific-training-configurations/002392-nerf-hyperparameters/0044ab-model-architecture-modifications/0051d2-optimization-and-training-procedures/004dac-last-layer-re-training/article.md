# Last Layer Re-training

Also called: Instant NeRF Editing, Last Layer Fine-tuning

## Overview
Last Layer Re-training is a technique for instantly propagating a color edit from a single user-provided view to an entire pre-trained Neural Radiance Field (NeRF) by selectively retraining only the final layer of the color Multi-Layer Perceptron (MLP) [S1]. It addresses the computational cost and view-dependent inconsistency issues of full model retraining, preserving existing geometric and view-dependent information while enabling near real-time editing [S1]. Unlike full retraining, which risks altering useful pre-trained features, this method restricts parameter updates to the last layer, ensuring rapid convergence and higher-quality edits [S1].

## How it works
**Inputs and Outputs:**
*   **Input:** A pre-trained NeRF model (specifically the color MLP $MLP_c$) and a single edited view image $I_{edit} \in \mathbb{R}^{W \times H \times 3}$ [S1].
*   **Output:** A new set of parameters $\phi'_{last}$ for the last layer of the color MLP, resulting in an edited NeRF [S1].

**Data Structures:**
The color MLP $MLP_c$ takes as input the concatenation of multiresolution hash grid features $f_x$ and a view direction encoding $\gamma(\theta)$ [S1].
The feature encoding $f_x$ is constructed by concatenating features from $L$ resolution levels:
$$
f_x = \{f_{x,0} \oplus f_{x,1} \oplus \dots \oplus f_{x,L}\}
$$
where $\oplus$ denotes concatenation [S1]. The MLP structure consists of hidden layers and a final output layer. The penultimate layer output is denoted as $h_{x,\theta}$, which depends on position $x$ and view direction $\theta$ [S1].

**Algorithm:**
1.  **Freeze Pre-trained Weights:** Keep all weights in the color MLP $MLP_c$ except for the final layer fixed. Let $\phi_c$ be the original weights and $\phi'_{last}$ be the trainable weights for the last layer [S1].
2.  **Define Forward Pass:** Compute the predicted color $c'_{x,\theta}$ using the penultimate layer output $h_{x,\theta}$ and the new last layer weights $\phi'_{last}$:
    $$
    MLP_{last}(h_{x,\theta}, \phi'_{last}) = c'_{x,\theta}
    $$
    where $h_{x,\theta}$ is the output of the penultimate layer of the pre-trained color MLP [S1].
3.  **Compute Loss:** Minimize the Mean Squared Error (MSE) between the volume-rendered RGB value $C'_{ij}$ from the edited NeRF and the original RGB value $C_{ij}$ from the training image (the edited view) [S1]:
    $$
    L = \sum_{(i,j) \in \text{pixels}} \| C_{ij} - C'_{ij} \|^2
    $$
4.  **Optimize:** Update $\phi'_{last}$ using gradient descent. The standard configuration uses the Adam optimizer with a constant learning rate [S1].
5.  **Iterate:** Repeat steps 2-4 for a fixed number of iterations [S1].

**Parameters:**
*   **Optimizer:** Adam [S1].
*   **Learning Rate:** 0.01 [S1].
*   **Iterations:** 200 iterations per scene [S1].
*   **Hardware:** Tested on NVIDIA RTX 3090 GPU [S1].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def last_layer_retraining(pretrained_color_mlp, edited_image, num_iterations=200, lr=0.01):
    """
    Retrains only the last layer of the color MLP to match the edited image.
    
    Args:
        pretrained_color_mlp: The pre-trained color MLP model.
        edited_image: The single user-edited view image (RGB).
        num_iterations: Number of optimization steps (default 200).
        lr: Learning rate for Adam optimizer (default 0.01).
        
    Returns:
        new_last_layer_weights: The updated weights for the last layer.
    """
    # Extract the last layer parameters from the pre-trained model
    last_layer = pretrained_color_mlp.last_layer
    penultimate_output_fn = pretrained_color_mlp.penultimate_layer_forward
    
    # Initialize trainable parameters for the last layer
    # Note: In practice, this involves creating a new parameter tensor 
    # initialized with the pre-trained weights and marking it as requires_grad
    new_last_layer_weights = last_layer.weight.clone().detach().requires_grad_(True)
    new_last_layer_bias = last_layer.bias.clone().detach().requires_grad_(True)
    
    # Define optimizer for the last layer only
    optimizer = Adam([{'params': [new_last_layer_weights, new_last_layer_bias]}], lr=lr)
    
    for iteration in range(num_iterations):
        optimizer.zero_grad()
        
        # 1. Sample rays through the camera for the edited image
        # This step depends on the specific ray sampling implementation of the NeRF.
        # For each pixel (i, j), we get a ray origin and direction.
        # We then sample points x along the ray and compute view directions theta.
        
        # 2. Get penultimate layer outputs for all sampled points
        # h_x_theta is the output of the penultimate layer for position x and view theta
        # This part uses the frozen pre-trained weights.
        h_x_theta = penultimate_output_fn(positions=x, view_directions=theta)
        
        # 3. Apply the new last layer weights to predict color
        # c_prime is the predicted color using the new last layer weights
        c_prime = Linear(new_last_layer_weights, new_last_layer_bias)(h_x_theta)
        
        # 4. Volume render the predicted color to get C'_ij
        # This involves alpha compositing along the ray.
        # C_prime_ij is the final rendered RGB value for pixel (i, j)
        C_prime_ij = volume_render(c_prime, alphas=alpha, depths=depth)
        
        # 5. Compute loss against the edited image
        # C_ij is the ground truth RGB from the edited image
        C_ij = get_pixel_color(edited_image, i, j)
        loss = MSE(C_prime_ij, C_ij)
        
        # 6. Backpropagate and update only the last layer weights
        loss.backward()
        optimizer.step()
        
    return new_last_layer_weights, new_last_layer_bias
```

**Edge Cases and Pitfalls:**
*   **Color Bleeding:** Since the information in the color MLP is global, modifying the last layer may cause edits to propagate to undesired regions of the scene [S1]. This is addressed in IReNe by introducing a lightweight 3D segmentation step to restrict edits to target regions [S1].
*   **View-Dependent Effects:** Editing only the last layer must preserve view-dependent effects (e.g., specularities) present in the pre-trained model. The method relies on the pre-trained penultimate layer to encode this view-dependent information [S1].

## Subtopics

### [IReNe](13549-irene.md)
A specific implementation of last layer re-training for instant 360-degree recoloring. It prevents color bleeding at object boundaries and ensures multi-view consistency for view-dependent effects by selectively fine-tuning only the diffuse neurons in the last layer. It achieves high-quality edits in under 5 seconds, outperforming methods like ICE-NeRF (color bleeding) and Palette-NeRF (slow training).

## Operating characteristics
*   **Time Complexity:** Significantly faster than full retraining because only the last layer parameters are updated. IReNe achieves edits in under 5 seconds [S1].
*   **Memory Complexity:** Low memory overhead compared to full retraining, as only the last layer weights need to be stored in a trainable state [S1].
*   **Performance:** Achieves higher-quality edits than full retraining in some cases, as it preserves useful pre-trained information [S1].
*   **Strengths:** Near real-time editing, preserves view-dependent effects, simple implementation.
*   **Limitations:** May cause color bleeding in global regions without additional segmentation (addressed by IReNe) [S1].
*   **When to use:** When instant color editing of a pre-trained NeRF is required, and preserving view-dependent effects is important.
*   **When not to use:** When the edit requires significant changes to the underlying geometry or density, or when the edit affects regions not covered by the single edited view without proper segmentation.

## Background
Last Layer Re-training was introduced in the IReNe paper (Mazzucchelli et al., CVPR 2024) as a method for instant recoloring of Neural Radiance Fields [S1]. It builds upon the Instant-NGP architecture, utilizing its multiresolution hash grid features and MLP structure [S1]. The method was developed to address the limitations of prior editing methods, which were either slow (Palette-NeRF) or suffered from visual artifacts like color bleeding (ICE-NeRF) [S1].

## Sources
[S1] IReNe: Instant Recoloring of Neural Radiance Fields, https://openaccess.thecvf.com/content/CVPR2024/papers/Mazzucchelli_IReNe_Instant_Recoloring_of_Neural_Radiance_Fields_CVPR_2024_paper.pdf, fetched 2026-10-29
