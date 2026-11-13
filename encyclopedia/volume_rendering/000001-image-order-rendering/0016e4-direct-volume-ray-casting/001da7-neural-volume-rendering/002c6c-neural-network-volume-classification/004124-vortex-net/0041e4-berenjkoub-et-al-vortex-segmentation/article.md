# Berenjkoub et al. vortex segmentation
Also called: Vortex segmentation using synthetic cylinder flows

## Overview
A method that trains convolutional neural networks on synthetic cylinder flows with analytically known vortex boundaries to identify vortices in volumetric data, eliminating the need for manual threshold tuning associated with traditional criteria like $\lambda_2$ [S1, unofficial]. It sets itself apart from alternative criteria-based approaches by bypassing manual threshold selection entirely, relying instead on direct supervised learning from parametrically generated ground truth [S1, unofficial].

## How it works
**Inputs and Outputs:**
- Input: $64 \times 64$ 2D flow patches [S1, unofficial].
- Output: Binary segmentation mask indicating vortex presence [S1, unofficial].

**Data Structures:**
- Synthetic Cylinder Flows: Parametrically generated 2D velocity fields where the vortex boundary is defined analytically [S1, unofficial].
- Patches: Extracted $64 \times 64$ sub-regions from the 2D flow fields [S1, unofficial].
- Ground Truth: Binary labels derived directly from the analytical boundary definition, requiring no thresholding [S1, unofficial].

**Steps:**
1. Generate synthetic cylinder flows with known analytical vortex boundaries [S1, unofficial].
2. Extract $64 \times 64$ patches of 2D flow [S1, unofficial].
3. Train CNN, ResNet, and U-Net architectures on these patches [S1, unofficial].
4. Evaluate F-score [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def generate_synthetic_cylinder_flow(patches_per_flow, patch_size=64):
    """Step 1 & 2: Generate synthetic cylinder flows and extract 64x64 patches [S1, unofficial]."""
    data = []
    for _ in range(patches_per_flow):
        flow_field = generate_parametric_flow() # [G] Standard parametric flow generation
        for _ in range(100):
            x, y = random_patch_coordinates(flow_field.shape) # [G] Standard random coordinate sampling
            patch = flow_field[x:x+patch_size, y:y+patch_size]
            mask = compute_analytical_vortex_mask(patch) # [G] Binary mask derived from analytical boundary
            data.append((patch, mask))
    return data

def train_segmentation_models(data):
    """Step 3: Train CNN, ResNet, and U-Net architectures on these patches [S1, unofficial]."""
    models = {'CNN': build_cnn(), 'ResNet': build_resnet(), 'U-Net': build_unet()} # [G] Standard architecture definitions
    results = {}
    for name, model in models.items():
        model.compile(loss='binary_crossentropy', optimizer='adam') # [G] Standard training setup
        patches, masks = zip(*data)
        model.fit(patches, masks, epochs=100, batch_size=32) # [G] Standard training loop
        predictions = model.predict(patches)
        f_score = calculate_f_score(masks, predictions)
        results[name] = f_score
    return results

def calculate_f_score(true_masks, pred_masks):
    """Step 4: Evaluate F-score [S1, unofficial]."""
    tp = fp = fn = 0
    for true, pred in zip(true_masks, pred_masks):
        for t, p in zip(true, pred):
            t_bin = 1 if t > 0.5 else 0
            p_bin = 1 if p > 0.5 else 0
            if p_bin == 1 and t_bin == 1: tp += 1
            elif p_bin == 1 and t_bin == 0: fp += 1
            elif p_bin == 0 and t_bin == 1: fn += 1
    precision = tp / (tp + fp + 1e-8)
    recall = tp / (tp + fn + 1e-8)
    return 2 * (precision * recall) / (precision + recall + 1e-8)
```

**Parameters:**
- Patch Size: $64 \times 64$ pixels [S1, unofficial].
- Architectures: CNN, ResNet, U-Net [S1, unofficial].
- Metric: F-score [S1, unofficial].

**Edge Cases and Pitfalls:**
- Validation was performed on a single 2D flow configuration, leaving open the question of transfer to 3D turbulent fields [S1, unofficial].

## Subtopics

### Bai et al. 3D Vortex Segmentation
A method that operates directly on 3D grids using an MLP for local velocity vector classification to identify vortices and flow regimes, avoiding the memory overhead of 3D convolutions [S1, unofficial]. It differs from Berenjkoub et al. by handling 3D volumetric data natively rather than relying on 2D patches, and by including a separate classification network for flow regimes based on the relationship between local vorticity magnitude and Reynolds number [S1, unofficial]. It should be chosen when 3D context is critical and computational resources for 3D convolutions are constrained [S1, unofficial]. See [Bai et al. 3D Vortex Segmentation](10429-vortex-net.md#bai-et-al-3d-vortex-segmentation).

## Operating characteristics
- **Time Complexity:** Not given in the sources.
- **Memory Complexity:** Not given in the sources.
- **Measured Performance:** U-Net achieved the highest F-score of 0.997 on the synthetic 2D cylinder flow data [S1, unofficial].
- **Strengths:** Eliminates the need for manual threshold tuning, which is a bottleneck for criteria like $\lambda_2$ [S1, unofficial]. U-Net's skip connections preserve boundary detail effectively [S1, unofficial].
- **Limitations:** Validated only on a single 2D flow configuration; transfer to 3D turbulent fields is unverified [S1, unofficial].
- **When to use:** When precise vortex boundary identification is required and manual threshold tuning is undesirable, particularly for 2D flow analysis [S1, unofficial].
- **When not to use:** For 3D turbulent fields where transferability has not been established, or when computational resources for 3D convolutions are not a constraint and 3D context is critical (consider [Bai et al. 3D Vortex Segmentation](10429-vortex-net.md#bai-et-al-3d-vortex-segmentation) instead) [S1, unofficial].

## Background
The method was introduced by Berenjkoub et al. [9] as a response to the bottleneck of threshold selection in vortex identification criteria like $\lambda_2$, which often require manual adjustment to match visual expectations [S1, unofficial]. It builds on the idea of using synthetic data with known ground truth to train neural networks, a concept also explored in [Vortex-Net](10429-vortex-net.md) but applied here to bypass criterion-specific boundaries entirely [S1, unofficial]. The approach was validated against CNN, ResNet, and U-Net architectures [S1, unofficial].

## Sources
[S1] Deep Learning Methods and Neural Representations for Scientific Volume ..., https://www.cs.uoregon.edu/Reports/AREA-202603-Mathai.pdf, fetched 2026-10-27 (unofficial)
