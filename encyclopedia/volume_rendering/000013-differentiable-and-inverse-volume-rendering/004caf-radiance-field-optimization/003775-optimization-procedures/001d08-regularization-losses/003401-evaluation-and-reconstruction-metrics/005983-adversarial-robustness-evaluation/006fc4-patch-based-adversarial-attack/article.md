# Patch-based Adversarial Attack

Also called: Adversarial Patch Attack

## Overview
The Patch-based Adversarial Attack is a method for evaluating the robustness of neural radiance field reconstruction networks by introducing distortions confined to a region of limited area (a patch) with unbounded intensity values, rather than applying global low-intensity noise [S1]. It solves the problem of assessing vulnerability to localized, high-magnitude perturbations that may correspond to physical objects (e.g., stickers) in real-world scenarios [S1]. Unlike global perturbation methods, this approach exploits the fact that patches can manipulate distant regions of the rendered output while keeping original output values elsewhere, with position and view angle having little impact on efficacy [S1].

## How it works
**Inputs and Outputs:**
- **Input:** A set of $N$ source images (views) for a scene. The attacker selects a subset of these views to attack.
- **Output:** Modified source images containing adversarial patches.

**Data Structures:**
- **Patch:** Defined by its spatial location (center coordinates), size (width $w$, height $h$), and pixel intensity values.
- **Optimization Variables:** The pixel values within the patch boundaries. Unlike low-intensity attacks where perturbations are bounded by $\epsilon$, patch intensities are unbounded [S1].

**Parameters:**
- **Patch Size:** The most crucial parameter, analogous to the maximal change amount in low-intensity attacks. Tested sizes include $2\times2$, $5\times5$, $10\times10$, and $20\times20$ pixels [S1].
- **Number of Attacked Views:** The subset of source images receiving patches. Experiments used 2, 4, or 10 out of 10 views [S1].

**Steps:**
1. **Initialization:** For each source image selected for attack, initialize a patch at the center of the image. This placement ensures the patch is not closely positioned to regions already modified by previous attacks or scene content [S1].
2. **Independent Optimization:** Optimize the pixel values within each patch independently for each source image. This allows pixel values at the same spatial location to differ across different images, exploiting individual view vulnerabilities [S1].
3. **Loss Computation:** Compute the loss function based on the difference between the rendered output of the attacked NeRF and the target adversarial ground truth (e.g., hallucinated objects, deleted objects) [S1].
4. **Gradient Update:** Update the patch pixel values using gradient descent to minimize the loss. The unbounded nature of the intensities allows for arbitrary values to be introduced [S1].
5. **Iteration:** Repeat steps 3-4 until convergence or a maximum number of iterations is reached.

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def patch_based_adversarial_attack(source_images, target_output, patch_size, num_iterations):
    """
    source_images: List of N source images (tensors)
    target_output: Target rendered output (tensor)
    patch_size: Tuple (width, height) of the patch
    num_iterations: Number of optimization steps
    """
    attacked_images = []
    
    for i, image in enumerate(source_images):
        # Initialize patch at center of image with random or zero values
        # Assuming image dimensions are H x W
        h, w = image.shape[1], image.shape[2]
        patch_h, patch_w = patch_size
        
        # Calculate center coordinates
        center_y, center_x = h // 2, w // 2
        
        # Create a mask for the patch region
        y_start = max(0, center_y - patch_h // 2)
        y_end = min(h, center_y + patch_h // 2)
        x_start = max(0, center_x - patch_w // 2)
        x_end = min(w, center_x + patch_w // 2)
        
        # Extract patch region
        patch = image[:, y_start:y_end, x_start:x_end].clone()
        patch.requires_grad = True
        
        optimizer = torch.optim.Adam([patch], lr=0.1)
        
        for _ in range(num_iterations):
            optimizer.zero_grad()
            
            # Create modified image with patch
            modified_image = image.clone()
            modified_image[:, y_start:y_end, x_start:x_end] = patch
            
            # Render the modified image using the NeRF
            # Note: In practice, this involves differentiable rendering
            rendered_output = render_neRF(modified_image)
            
            # Compute loss against target output
            loss = compute_loss(rendered_output, target_output)
            
            if loss.requires_grad:
                loss.backward()
                optimizer.step()
        
        # Update the image with the optimized patch
        attacked_image = image.clone()
        attacked_image[:, y_start:y_end, x_start:x_end] = patch.detach()
        attacked_images.append(attacked_image)
        
    return attacked_images
```

**Edge Cases and Pitfalls:**
- **Real-world Deviation:** The experimental setup deviates from real-world applications because patches are consistently centered and fixed regardless of viewpoint, and pixel values vary independently across input images rather than representing a physical object [S1].
- **Patch Size Sensitivity:** If the patch size is too small, the attack may fail to significantly impact the rendered output [S1].

## Subtopics

### [Fast Gradient Sign Method (FGSM)](21941-fast-gradient-sign-method-fgsm.md)
The Fast Gradient Sign Method (FGSM) is a single-step adversarial attack that generates perturbations by moving the input in the direction of the sign of the gradient of the loss function with respect to the input. It solves the problem of efficiently creating adversarial examples by approximating the steepest ascent direction using only the sign of the gradient, allowing for rapid attack generation. Unlike iterative methods, FGSM computes the perturbation in a single pass, prioritizing speed over the precision of the perturbation magnitude.

### [Adversarial Attack on GeNeRF](21975-adversarial-attack-on-generf.md)
A targeted adversarial attack method for Generalizable Neural Radiance Fields (GeNeRFs) that modifies source images (views) to force the rendered output to match a manually created 'adversarial ground truth' image containing hallucinated objects, deleted objects, or modified shapes. Unlike standard classification attacks that perturb a single image, this method exploits the multi-view nature of GeNeRFs by simultaneously modifying a subset or all of the input source views. It distinguishes itself from sibling methods like [Fast Gradient Sign Method (FGSM)](21941-fast-gradient-sign-method-fgsm.md) by integrating a residual loss regularizer that preserves image quality in non-attacked regions, ensuring gradients from unmodified areas are considered during optimization.

### Low-intensity adversarial attack
**Parameter:**
- $\epsilon$ value: The crucial algorithm parameter intended to ensure the challenging detectability of the adversarial modifications [S1].

**Metrics:**
- Attack success metrics for 2/10 attacked views: L2=0.022, PSNR=19.75, SSIM=0.537, LPIPS=0.242.
- Attack success metrics for 4/10 attacked views: L2=0.016, PSNR=21.83, SSIM=0.841, LPIPS=0.168.
- Attack success metrics for 10/10 attacked views: L2=0.011, PSNR=24.72, SSIM=0.910, LPIPS=0.163.

**Comparison:**
- Attack effectiveness diminishes when multiple views are utilized and not all source images are accessible to the attacker, but increases significantly when the attacker has access to the majority of views [S1].

**Usage:**
- Evaluated on GeNeRF across 10 scenes, using 10 source views per scene. Metrics calculated as the average of 10 independent runs [S1].

### Carlini-Wagner Attack
**Definition:**
- An attack approach based on binary classifier robustness, calculating the distance to the separating hyperplane and deriving a closed-form formula for the smallest perturbation to change the classifier's output, applied iteratively [S1].

**Formula:**
- $\Delta(x_0; f)$ represents the distance from point $x_0$ to the separating hyperplane [S1].

**History:**
- Proposed by Moosavi et al. [29] [S1].

### Black-Box Attack
**Definition:**
- An attack method where the attacker only has access to final responses or confidence values, using evolutionary algorithms to generate attacks without needing network gradients [S1].

**History:**
- Referenced in [33], [34] [S1].

## Operating characteristics
**Time and Memory Complexity:**
- Not given in the sources.

**Measured Performance:**
- For a $20\times20$ patch size, the attack success metrics are:
    - 2/10 attacked views: L2=0.022, PSNR=18.43, SSIM=0.588, LPIPS=0.256.
    - 4/10 attacked views: L2=0.017, PSNR=21.60, SSIM=0.792, LPIPS=0.173.
    - 10/10 attacked views: L2=0.011, PSNR=24.33, SSIM=0.903, LPIPS=0.114.
- These measurements were calculated on the average of 10 independent runs and conducted on ten different scenes [S1].

**Scaling:**
- Attack effectiveness increases significantly when the attacker has access to the majority of the views [S1].

**Strengths and Limitations:**
- **Strengths:** Patches are not restricted to local neighborhoods; they can manipulate distant regions while keeping original output values elsewhere. Position and view angle have little impact on efficacy [S1].
- **Limitations:** The experimental setup deviates from real-world applications because patches are consistently centered and fixed regardless of viewpoint, and pixel values vary independently across input images rather than representing a physical object [S1].

**When to Use:**
- When evaluating the robustness of neural radiance field reconstruction networks against localized, high-magnitude perturbations that may correspond to physical objects in real-world scenarios [S1].

**When Not to Use:**
- When the goal is to simulate real-world physical attacks where patches must maintain consistent appearance across viewpoints and represent a physical object [S1].

## Background
The patch-based adversarial attack was presented in [16], [31]. It was also demonstrated with black and white patches in [32], where not the intensities of the patch, but the locations and sizes of the stickers are optimized [S1]. These attacks, where the gradients of the networks are not necessarily used during optimization, open space towards black-box attacks [33], [34] [S1]. The resilience of segmentation networks against adversarial attacks was investigated heavily in the past years [36], [37], [38], [39] [S1]. Subsequent years witnessed extensive investigations into the potential of exploiting adversarial attacks, with researchers developing novel attack strategies to enhance the robustness of generated attacks [16], [31] [S1].

## Sources
[S1] Targeted Adversarial Attacks on Generalizable Neural Radiance Fields, https://doi.org/10.1109/iccvw60793.2023.00399, fetched 2026-10-29
