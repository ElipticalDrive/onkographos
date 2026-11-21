# Instruct-NeRF2NeRF Optimization

Also called: Instruct-NeRF2NeRF optimization process, NeRF Editing via Diffusion Guidance

## Overview
Instruct-NeRF2NeRF is a method for editing pre-trained Neural Radiance Fields (NeRFs) using natural language instructions. It solves the problem of modifying 3D scene appearance and geometry without retraining from scratch by leveraging a 2D text-to-image diffusion model as a guidance signal [S1]. Unlike sibling methods that may use semantic segmentation or geometric constraints for inpainting, this approach relies on InstructPix2Pix to generate edited views, which are then backpropagated to update the NeRF parameters [S1]. It is distinct in its use of classifier-free guidance weights to control edit strength and its lack of a single convergence point due to the stochastic nature of the diffusion model [S1].

## How it works

### Inputs and Outputs
**Inputs:**
1.  A pre-trained NeRF model $f_\theta: (\mathbf{x}, \mathbf{d}) \to (\sigma, \mathbf{c})$, typically initialized using the 'nerfacto' model from NeRFStudio trained for 30,000 iterations [S1].
2.  A set of input images used for the initial NeRF training.
3.  A natural language edit instruction (e.g., "turn the man into a bear") [S1].
4.  Classifier-free guidance weights for the diffusion model: $s_I$ (image guidance) and $s_T$ (text guidance) [S1].

**Outputs:**
1.  An edited NeRF model with updated parameters $\theta$ that reflects the instruction [S1].

### Data Structures
The method operates on the standard NeRF representation. No new complex data structures are introduced in the core loop, but the implementation relies on:
*   **NeRF Parameters $\theta$:** Stored in the NeRFStudio optimizer state. These must be re-initialized before the editing loop begins [S1].
*   **Diffusion Model State:** The InstructPix2Pix model weights and the Stable Diffusion autoencoder state [S1].

### Numbered Steps

1.  **Initialization:** Load the pre-trained NeRF. Re-initialize the optimizers in the NeRF training framework to ensure gradients are computed correctly for the editing phase [S1].
2.  **View Selection:** For each optimization iteration, select a single camera view (position $\mathbf{x}$ and direction $\mathbf{d}$) from the training dataset or a random sampling distribution [S1].
3.  **Rendering:** Render the image $\mathbf{I}_{render}$ from the current NeRF $f_\theta$ at the selected view [S1].
4.  **Diffusion Editing:** Pass $\mathbf{I}_{render}$, the edit instruction, and guidance weights $s_I, s_T$ to the InstructPix2Pix model. Generate the edited image $\mathbf{I}_{edit}$ [S1].
    *   This step uses the public implementation from the Diffusers library [S1].
5.  **Loss Computation:** Compute the photometric loss between the rendered image and the edited image:
    $$
    L_{render} = \| \mathbf{I}_{render} - \mathbf{I}_{edit} \|^2
    $$
    [S1]
6.  **Backpropagation:** Backpropagate the loss through the NeRF to update the parameters $\theta$. The gradient flows from the edited image back to the NeRF weights [S1].
7.  **Iteration:** Repeat steps 2–6 for a specified number of iterations.
8.  **Termination:** Stop optimization at a subjective iteration count. There is no automatic convergence criterion [S1].

### Parameters and Typical Values
*   **$s_I$ (Image Classifier-Free Guidance):** Controls the influence of the input image on the edit.
    *   Typical range: $1.3$ to $1.75$ [S1].
    *   Specific examples: $1.5$ (Tree, Tents, Person, Fig 1, Bear), $1.3-1.5$ (Fig 7) [S1].
*   **$s_T$ (Text Classifier-Free Guidance):** Controls the influence of the text instruction.
    *   Typical range: $6.5$ to $12.5$ [S1].
    *   Specific examples: $7.5-12.5$ (Tree), $10.0$ (Tents), $6.5-7.5$ (Person, Fig 1, Bear), $6.5-8.5$ (Fig 7) [S1].
*   **Training Iterations:**
    *   Small scenes (<100 input images): $3,000$ to $4,000$ iterations [S1].
    *   Large scenes (>200–300 input images): $7,000$ to $8,000$ iterations [S1].

### Pseudocode

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def instruct_nerf2nerf_edit(nerf_model, optimizer, input_images, instruction, s_I, s_T, max_iterations):
    """
    Edits a NeRF using InstructPix2Pix guidance.
    
    Args:
        nerf_model: Pre-trained NeRF model f_theta.
        optimizer: NeRF optimizer (re-initialized before call).
        input_images: List of original training images (used for view sampling).
        instruction: String edit instruction.
        s_I: Image guidance weight.
        s_T: Text guidance weight.
        max_iterations: Maximum number of optimization steps.
        
    Returns:
        nerf_model: Edited NeRF model.
    """
    # Re-initialize optimizers as per source [S1]
    optimizer.reset()
    
    # InstructPix2Pix model loaded from Diffusers library [S1]
    # model = load_instruct_pix2pix() 
    
    for step in range(max_iterations):
        # 1. Select a single view
        # In practice, sample a random index from input_images
        idx = random.randint(0, len(input_images) - 1)
        camera_params = get_camera_params_for_image(idx, input_images)
        
        # 2. Render current NeRF view
        # render() returns color tensor C and depth D
        C_render, D_render = nerf_model.render(camera_params)
        
        # 3. Edit the rendered image using InstructPix2Pix
        # I_edit = InstructPix2Pix(I_render, instruction, s_I, s_T)
        I_edit = call_instruct_pix2pix(C_render, instruction, s_I, s_T)
        
        # 4. Compute Loss
        # L = || I_render - I_edit ||^2
        loss = torch.mean((C_render - I_edit) ** 2)
        
        # 5. Backpropagate
        optimizer.zero_grad()
        loss.backward()
        optimizer.step()
        
        # Note: Termination is subjective based on iteration count [S1]
        if step % 100 == 0:
            print(f"Step {step}, Loss: {loss.item()}")
            
    return nerf_model
```

### Edge Cases and Pitfalls
*   **Lack of Convergence:** The diffusion model produces temporally varying images. The edit type and strength change over training, so there is no single convergence point. The user must subjectively choose the termination iteration [S1].
*   **Texture Blurriness:** Using the Stable Diffusion autoencoder on input images causes gradual texture blurriness. The autoencoder creates locally similar but non-3D-consistent textures, leading to blurry renders in the final reconstruction [S1].
*   **Double Faces:** Single-view diffusion guidance can cause artifacts such as double faces on added objects [S1].
*   **View Inconsistency:** Edit instructions may only be relevant to certain camera views. For example, if the instruction is "turn the man into a bear," views not prominently featuring the man may produce less or no edit, leading to inconsistency [S1].
*   **Spatial Manipulation Limits:** The method inherits limitations from InstructPix2Pix, including inability to perform large spatial manipulations or binding between objects [S1].

## Operating characteristics

### Time and Memory Complexity
*   **Time:** Each iteration requires one NeRF forward pass (rendering) and one diffusion model forward pass (InstructPix2Pix). The diffusion model is significantly more expensive than the NeRF rendering. Total time scales linearly with the number of iterations (3,000–8,000) and the complexity of the diffusion model [S1].
*   **Memory:** Requires memory for the NeRF parameters, the optimizer state, and the entire InstructPix2Pix/Stable Diffusion model in VRAM [S1].

### Measured Performance
*   **Small Scenes (<100 images):** Converges (subjectively) in 3,000–4,000 iterations [S1].
*   **Large Scenes (>200-300 images):** Converges (subjectively) in 7,000–8,000 iterations [S1].
*   **Quality:** Edited scenes often contain slightly blurrier textures compared to the original NeRF due to the autoencoder effects [S1].

### Strengths and Limitations
*   **Strengths:** Allows editing via natural language without 3D annotations. Simple to implement using existing NeRFStudio and Diffusers libraries [S1].
*   **Limitations:** Cannot perform large spatial manipulations. Struggles with object binding. Single-view guidance leads to multi-view inconsistencies (e.g., double faces). Quality degrades over long optimization runs [S1].

### When to Use
*   When natural language editing is required.
*   When the scene is relatively small or moderate in size.
*   When the user can tolerate subjective termination and potential blurriness.

### When Not to Use
*   When strict multi-view consistency is required without post-processing.
*   When large spatial transformations (e.g., moving an object from one side of the room to another) are needed.
*   When high-fidelity texture preservation is critical (due to autoencoder blurring) [S1].

## Background

### Origin
The method was introduced by [S1] Zhizhuo Zhou and Shubham Tulsiani in the paper "Instruct-NeRF2NeRF: Editing 3D Scenes with Instructions" [S1]. It builds upon the InstructPix2Pix model [S1] and the general framework of differentiable rendering.

### History and Development
The method leverages the Stable Diffusion autoencoder and InstructPix2Pix implementation from the Diffusers library [S1]. It was developed to address the limitations of previous NeRF editing methods that required complex 3D priors or manual segmentation. The authors noted that the diffusion guidance produces temporally varying images, distinguishing it from standard NeRF optimization which has a clear convergence point [S1].

### Usage
It is used for 3D scene editing where users provide text instructions to modify the appearance or content of a NeRF reconstruction [S1].

## Sources

[S1] Instruct-NeRF2NeRF: Editing 3D Scenes with Instructions, https://instruct-nerf2nerf.github.io/, fetched 2026-10-28
