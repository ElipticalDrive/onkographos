# View-Dependent Effects Preservation

Also called: Neuron Freezing for View-Dependence

## Overview
View-Dependent Effects Preservation is a technique used during the editing of pre-trained Neural Radiance Fields (NeRFs) to retain existing specularities and other view-dependent appearance effects that would otherwise be lost or distorted by global recoloring operations. It distinguishes itself from standard NeRF editing methods by automatically identifying and freezing specific neurons in the final color prediction layer that encode angular variation, thereby isolating geometry information from view-dependent appearance [S1].

## How it works
**Inputs and Outputs**
*   **Input:** A pre-trained NeRF model (specifically the color MLP $V_{rgb}$), an input edited view image $I_{edit}$, and scene parameters (azimuth sweep range).
*   **Output:** A mask or set of indices identifying which neurons in the last layer of the color MLP are "view-dependent" and should be frozen during subsequent training.

**Data Structures**
*   **Neuron Activations ($h_\theta$):** A tensor of shape $\mathbb{R}^{W \times H \times 64}$, where $W$ and $H$ are the image dimensions, and 64 is the dimensionality of the output vector from the penultimate layer of the color MLP.
*   **Angular Profile ($A$):** A matrix of shape $\mathbb{R}^{64 \times Q}$, where $Q$ is the number of sampled view angles. Each row $A_k \in \mathbb{R}^Q$ represents the activation profile of neuron $k$ across the sampled views.

**Parameters**
*   $\tau_1$: Threshold for the coefficient of variation. Default value: $0.5$ [S1].
*   $\tau_2$: Threshold for minimum activation magnitude. Default value: $100$ [S1].
*   $Q$: Number of view angles to sample. Default value: $30$ [S1].
*   $\psi$: Angular sweep step. For omnidirectional scenes (e.g., Blender, MIP360), the full azimuth is swept. For forward-facing scenes (e.g., LLFF), only half the azimuth is considered [S1].

**Algorithm Steps**
1.  **Sample View Directions:** Generate a set of $Q$ view directions $\{\theta_0, \theta_1, ..., \theta_{Q-1}\}$ by uniformly sweeping the azimuth angle according to $\psi$. For forward-facing datasets, restrict the sweep to 180 degrees; for others, sweep 360 degrees [S1].
2.  **Compute Activations:** For each sampled view direction $\theta_i$, render the edited view $I_{edit}$ (or query the MLP at the corresponding pixel coordinates) to obtain the neuron activations $h_{\theta_i} \in \mathbb{R}^{W \times H \times 64}$ from the last layer of the color MLP [S1].
3.  **Aggregate Spatially:** For each neuron index $k \in \{0, ..., 63\}$, aggregate the activations across all spatial pixels $(w, h)$ and all view angles to form the angular profile $A_k$. Specifically, $A_k[i] = h_{\theta_i}[w, h, k]$ for all $w, h$. This results in $A \in \mathbb{R}^{64 \times Q}$ [S1].
4.  **Identify View-Dependent Neurons:** For each neuron $k$, calculate the mean $\mu(A_k)$ and standard deviation $\sigma(A_k)$ of its activation profile. Label neuron $k$ as view-dependent if either of the following conditions is met:
    *   The coefficient of variation is low: $\frac{\sigma(A_k)}{\mu(A_k)} < \tau_1$
    *   The minimum activation is low: $\min(A_k) \le \tau_2$
    
    *Note: The source text states "std(A_k) / mu(A_k) < \tau_1" as the condition for labeling a neuron as view-dependent. However, logically, a low coefficient of variation implies the neuron is *constant* (geometry-like), while a high variation implies it is view-dependent. The source text says: "We preserve much of the view-dependent information by freezing part of the weights... we identified two distinct behaviors: certain entries... exhibit significant changes... while others maintain constancy... We preserve... by freezing part of the weights... labeled as view-dependent if...". There is a contradiction in the source text's description of the threshold logic versus the intent. The source explicitly writes the formula for labeling as view-dependent as $\frac{std(A_k)}{\mu(A_k)} < \tau_1$. This implies that neurons with *low* variation are considered view-dependent in this specific context, or potentially the source contains a typo where the inequality is reversed or the label is swapped. Given the instruction to keep exact numbers and variable names, we reproduce the formula as cited, but note the potential logical inversion. Standard interpretation of "view-dependent" implies high variation. If the source intends to freeze view-dependent effects, it should freeze high-variance neurons. The source says "labeled as view-dependent if ... < \tau_1". We follow the source text exactly.* [S1].
5.  **Freeze Weights:** During the subsequent training phase for editing, the weights connecting the identified view-dependent neurons to the output are frozen, preserving their original values [S1].

**Pseudocode**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import numpy as np

def identify_view_dependent_neurons(nerf_model, edited_view_image, Q=30, tau1=0.5, tau2=100, is_forward_facing=False):
    """
    Identifies neurons in the last layer of the color MLP that exhibit view-dependent behavior.
    
    Args:
        nerf_model: Pre-trained NeRF model with a color MLP.
        edited_view_image: The input image used for editing (provides pixel coordinates).
        Q: Number of view angles to sample.
        tau1: Threshold for coefficient of variation.
        tau2: Threshold for minimum activation.
        is_forward_facing: Boolean indicating if the scene is forward-facing (LLFF).
        
    Returns:
        view_dependent_mask: Boolean array of shape (64,) indicating view-dependent neurons.
    """
    W, H = edited_view_image.shape[:2]
    num_neurons = 64
    
    # Determine azimuth sweep range
    if is_forward_facing:
        azimuth_range = np.linspace(0, np.pi, Q) # Half sweep
    else:
        azimuth_range = np.linspace(0, 2 * np.pi, Q) # Full sweep
        
    # Step 1 & 2: Compute activations for each view angle
    # h_theta shape: (Q, W, H, 64)
    h_theta = np.zeros((Q, W, H, num_neurons))
    
    for i, theta in enumerate(azimuth_range):
        # Query the MLP for the edited view at the current camera orientation
        # This requires rendering or querying the MLP for all pixels in the image
        # assuming the camera is positioned at azimuth theta.
        # The implementation details mention using Instant-NGP PyTorch impl.
        # Here we assume a function `query_mlp` that returns activations.
        h_theta[i] = query_mlp_activations(nerf_model, edited_view_image, theta)
        
    # Step 3: Aggregate along spatial dimensions
    # A shape: (64, Q)
    # We want A[k, i] to be the activation of neuron k at view angle i,
    # aggregated over all pixels. The source says "aggregate the values ... along spatial dimensions".
    # It does not specify mean or sum. However, to compute std and min of the profile,
    # we need a single value per neuron per view angle.
    # The source says "Ak in R^Q is a profile ... of each output color neuron k across the whole scene".
    # Usually, this implies aggregating over space. Let's assume mean aggregation for stability,
    # or simply collecting all values. The formula uses std(Ak) and min(Ak).
    # If Ak is R^Q, it must be one value per view angle.
    # "Aggregate the values of this set along spatial dimensions obtaining A in R^{64xQ}"
    # This implies reducing the (W,H) dimension. Mean is the most standard interpretation for "profile".
    
    A = np.zeros((num_neurons, Q))
    for k in range(num_neurons):
        # h_theta[:, :, :, k] has shape (Q, W, H)
        # Aggregate over W and H
        # Source doesn't specify mean/sum. We use mean as a representative value for the scene.
        # [G] Mean aggregation is assumed as it is standard for spatial reduction in this context.
        A[k, :] = np.mean(h_theta[:, :, :, k], axis=(1, 2))
        
    # Step 4: Label neurons
    view_dependent_mask = np.zeros(num_neurons, dtype=bool)
    
    for k in range(num_neurons):
        Ak = A[k, :]
        mu_Ak = np.mean(Ak)
        std_Ak = np.std(Ak)
        min_Ak = np.min(Ak)
        
        # Condition 1: Coefficient of variation < tau1
        # Handle division by zero if mu_Ak is 0
        if mu_Ak != 0:
            cv = std_Ak / mu_Ak
            cond1 = cv < tau1
        else:
            cond1 = False # Or handle as view-dependent if std is also 0? Source doesn't specify.
            
        # Condition 2: Min activation <= tau2
        cond2 = min_Ak <= tau2
        
        # Source: "labeled as view-dependent if either ... < tau1 OR ... <= tau2"
        if cond1 or cond2:
            view_dependent_mask[k] = True
            
    return view_dependent_mask

def query_mlp_activations(nerf_model, image, theta):
    """
    [G] Placeholder for the specific rendering/querying logic of the Instant-NGP implementation.
    This function would cast rays for each pixel in the image based on camera orientation theta
    and return the raw output of the penultimate layer of the color MLP.
    """
    # Implementation depends on the specific NeRF library (e.g., PyTorch Instant-NGP)
    pass
```

**Edge Cases and Pitfalls**
*   **Zero Mean:** If $\mu(A_k) = 0$, the coefficient of variation is undefined. The source does not specify handling for this case [S1].
*   **Forward-Facing Limitation:** The method assumes a specific angular sweep strategy for forward-facing scenes (LLFF) versus omnidirectional scenes. Using the wrong sweep range may misidentify view-dependent neurons [S1].
*   **Threshold Sensitivity:** The choice of $\tau_1$ and $\tau_2$ directly impacts which neurons are frozen. Incorrect values may freeze geometry-critical neurons or fail to freeze view-dependent ones [S1].

## Operating characteristics
*   **Time Complexity:** The method requires $Q$ forward passes through the color MLP for every pixel in the edited view. With $Q=30$ and typical image resolutions, this is computationally expensive but performed only once during the initialization of the editing process [S1].
*   **Memory Complexity:** Requires storing the activation tensor $h_\theta$ of size $\mathbb{R}^{Q \times W \times H \times 64}$ and the aggregated matrix $A$ of size $\mathbb{R}^{64 \times Q}$. For high-resolution images, this can be significant [S1].
*   **Strengths:** Preserves high-frequency view-dependent effects (specularities) without requiring explicit separation of diffuse and specular components during editing [S1].
*   **Limitations:** Relies on the assumption that view-dependent and geometry information are separable in the last layer's neurons, which may not hold for all NeRF architectures or scenes [S1].

## Background
The method was introduced in the context of **IReNe** (Instant Recoloring of Neural Radiance Fields) by Mazzucchelli et al. in 2024. It addresses the issue that global recoloring operations in NeRFs can distort existing view-dependent effects like specular highlights. The approach builds on the observation that the last layer of the color MLP contains neurons with distinct behaviors: some vary with view direction (view-dependent), while others remain constant (geometry information) [S1].

## Sources
[S1] IReNe: Instant Recoloring of Neural Radiance Fields, https://openaccess.thecvf.com/content/CVPR2024/papers/Mazzucchelli_IReNe_Instant_Recoloring_of_Neural_Radiance_Fields_CVPR_2024_paper.pdf, fetched 2026-10-29
