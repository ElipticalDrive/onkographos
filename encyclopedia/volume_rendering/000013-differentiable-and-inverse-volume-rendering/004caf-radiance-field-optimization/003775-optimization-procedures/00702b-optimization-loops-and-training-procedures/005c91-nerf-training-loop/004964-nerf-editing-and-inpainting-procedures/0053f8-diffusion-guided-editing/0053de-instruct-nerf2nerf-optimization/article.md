# Instruct-NeRF2NeRF Optimization

Also called: Instruct-NeRF2NeRF optimization process, NeRF Editing via Diffusion Guidance

## Overview
Instruct-NeRF2NeRF edits pre-trained Neural Radiance Fields (NeRFs) using natural language instructions by leveraging a 2D text-to-image diffusion model as a guidance signal [S1]. It modifies 3D scene appearance without retraining from scratch by rendering a single view, passing it to InstructPix2Pix along with a text instruction and classifier-free guidance weights to generate an edited image, and backpropagating the difference to update the NeRF parameters [S1]. Unlike sibling methods such as [RePaint-NeRF](14365-repaint-nerf.md) that use different priors, this approach relies on classifier-free guidance weights to control edit strength and lacks a single convergence point due to the temporally varying nature of diffusion guidance [S1].

## How it works

### Inputs and Outputs
**Inputs:**
1. A pre-trained NeRF model $f_\theta: (\mathbf{x}, \mathbf{d}) \to (\sigma, \mathbf{c})$, typically initialized using the 'nerfacto' model from NeRFStudio trained for 30,000 iterations [S1].
2. A set of input images used for the initial NeRF training [S1].
3. A natural language edit instruction (e.g., "turn the man into a bear") [S1].
4. Classifier-free guidance weights for the diffusion model: $s_I$ (image guidance) and $s_T$ (text guidance) [S1].

**Outputs:**
1. An edited NeRF model with updated parameters $\theta$ that reflects the instruction [S1].

### Data Structures
The method operates on the standard NeRF representation [S1]. The implementation relies on:
* **NeRF Parameters $\theta$:** Stored in the NeRFStudio optimizer state [S1].
* **Diffusion Model State:** The InstructPix2Pix model weights and the Stable Diffusion autoencoder state [S1].

### Numbered Steps
1. **Initialization:** Load the pre-trained NeRF [S1]. Re-initialize the optimizers in the NeRF training framework before starting the editing process [S1].
2. **View Selection:** For each optimization iteration, select a single camera view from the training dataset [S1].
3. **Rendering:** Render the image $\mathbf{I}_{\text{render}}$ from the current NeRF $f_\theta$ at the selected view [S1].
4. **Diffusion Editing:** Pass $\mathbf{I}_{\text{render}}$, the edit instruction, and guidance weights $s_I, s_T$ to InstructPix2Pix (via the Diffusers library) to generate the edited image $\mathbf{I}_{\text{edit}}$ [S1].
5. **Backpropagation:** Backpropagate the difference between the edited image and the rendered NeRF view to update the NeRF parameters [S1].
6. **Iteration:** Repeat steps 2–5 for a specified number of iterations [S1].
7. **Termination:** Stop optimization at a subjective iteration count based on scene size [S1].

### Formulas
* **Photometric Loss:** $L = \|\mathbf{I}_{\text{render}} - \mathbf{I}_{\text{edit}}\|^2$, where $\mathbf{I}_{\text{render}}$ is the rendered image and $\mathbf{I}_{\text{edit}}$ is the diffusion-generated edited image [S1].

### Parameters
* **$s_I$ (Image Classifier-Free Guidance):** Controls the influence of the input image on the edit [S1].
  * Typical range: $1.3$ to $1.75$ [S1].
  * Specific examples: $1.5$ (Tree, Tents, Person, Fig 1, Bear), $1.3$–$1.5$ (Fig 7) [S1].
* **$s_T$ (Text Classifier-Free Guidance):** Controls the influence of the text instruction [S1].
  * Typical range: $6.5$ to $12.5$ [S1].
  * Specific examples: $7.5$–$12.5$ (Tree), $10.0$ (Tents), $6.5$–$7.5$ (Person, Fig 1, Bear), $6.5$–$8.5$ (Fig 7) [S1].
* **Training Iterations:**
  * Small scenes (<100 input images): $3,000$ to $4,000$ iterations [S1].
  * Large scenes (>200–300 input images): $7,000$ to $8,000$ iterations [S1].

### Edge Cases and Pitfalls
* **Lack of Convergence:** Diffusion guidance produces temporally varying images during optimization, so the process lacks a single convergence point [S1]. The edit type and strength vary over training, requiring subjective termination [S1].
* **Texture Blurriness:** Using Stable Diffusion's autoencoder on input images before NeRF training causes gradual texture blurriness, as the autoencoder creates locally similar but non-3D-consistent textures [S1].
* **Double Faces:** Single-view diffusion guidance can cause artifacts like double faces on added objects [S1].
* **View Inconsistency:** Edit instructions may only be relevant to certain camera views, causing inconsistent or absent edits in other views [S1].
* **Spatial Manipulation Limits:** The method inherits limitations from InstructPix2Pix, including the inability to perform large spatial manipulations or binding between objects [S1].

### Pseudocode
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import random
import math

def instruct_nerf2nerf_edit(nerf_model, optimizer, camera_params_list, instruction, s_I, s_T, max_iterations):
    # Re-initialize optimizers in the NeRF training framework [S1]
    optimizer.reset()

    for step in range(max_iterations):
        # 1. Select a single view
        idx = random.randint(0, len(camera_params_list) - 1)
        cam = camera_params_list[idx]

        # 2. Render current NeRF view
        I_render = nerf_model.render(cam)

        # 3. Diffusion Editing: Pass I_render, instruction, and weights to InstructPix2Pix [S1]
        I_edit = _instruct_pix2pix_forward(I_render, instruction, s_I, s_T)

        # 4. Loss Computation
        loss = math.sqrt(sum((I_render[i] - I_edit[i]) ** 2 for i in range(len(I_render))))

        # 5. Backpropagation
        optimizer.zero_grad()
        loss.backward()
        optimizer.step()

    return nerf_model

def _instruct_pix2pix_forward(image, instruction, s_I, s_T):
    # Sources do not provide the InstructPix2Pix forward pass algorithm; uses Diffusers library implementation [S1]
    return image
```

## Operating characteristics

### Time and Memory Complexity
* **Time:** Each iteration requires one NeRF forward pass (rendering) and one diffusion model forward pass (InstructPix2Pix) [S1]. Total time scales with the number of iterations (3,000–8,000) [S1].
* **Memory:** Requires memory for the NeRF parameters, the optimizer state, and the InstructPix2Pix/Stable Diffusion model [S1].

### Measured Performance
* **Small Scenes (<100 images):** Subjectively converges in 3,000–4,000 iterations [S1].
* **Large Scenes (>200–300 images):** Subjectively converges in 7,000–8,000 iterations [S1].
* **Quality:** Edited scenes often contain slightly blurrier textures compared to the original NeRF due to autoencoder effects [S1].

### Strengths and Limitations
* **Strengths:** Allows editing via natural language without 3D annotations [S1]. Simple to implement using existing NeRFStudio and Diffusers libraries [S1].
* **Limitations:** Cannot perform large spatial manipulations [S1]. Struggles with object binding [S1]. Single-view guidance leads to multi-view inconsistencies (e.g., double faces) [S1]. Quality degrades over long optimization runs [S1].

### When to Use
* When natural language editing is required [S1].
* When the scene is relatively small or moderate in size [S1].
* When the user can tolerate subjective termination and potential blurriness [S1].

### When Not to Use
* When strict multi-view consistency is required without post-processing [S1].
* When large spatial transformations (e.g., moving an object from one side of the room to another) are needed [S1].
* When high-fidelity texture preservation is critical (due to autoencoder blurring) [S1].

## Background

### Origin
The method was introduced by Zhizhuo Zhou and Shubham Tulsiani in the paper "Instruct-NeRF2NeRF: Editing 3D Scenes with Instructions" [S1].

### History and Development
The method leverages the Stable Diffusion autoencoder and InstructPix2Pix implementation from the Diffusers library [S1]. It was developed to address the limitations of previous NeRF editing methods that required complex 3D priors or manual segmentation [S1]. The authors noted that the diffusion guidance produces temporally varying images, distinguishing it from standard NeRF optimization which has a clear convergence point [S1].

### Usage
It is used for 3D scene editing where users provide text instructions to modify the appearance or content of a NeRF reconstruction [S1].

## Sources

[S1] Instruct-NeRF2NeRF: Editing 3D Scenes with Instructions, https://instruct-nerf2nerf.github.io/, fetched 2026-10-28
