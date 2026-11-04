# PSNR Evaluation Pipeline

Also called: NeRF validation loop, PSNR calculation, Quantitative Evaluation, Test Set PSNR, Validation and PSNR Evaluation Pipeline

## Overview
The PSNR Evaluation Pipeline computes the Peak Signal-to-Noise Ratio (PSNR) on held-out test or validation splits to quantify the rendering quality of a trained NeRF model [S1, unofficial]. It differs from training metrics by using inference-optimized rendering paths and explicitly handling background compositing to ensure fair comparison against ground truth images [S2, unofficial].

## How it works
**Inputs:**
- Trained NeRF model state (coarse and/or fine networks) [S1, unofficial].
- Dataset split identifier: `"test"` or `"val"` [S1, unofficial].
- Scene metadata (camera intrinsics, near/far planes) [S1, unofficial].
- Ground truth images and poses for the specified split [S1, unofficial].

**Outputs:**
- Per-frame PSNR values.
- Mean PSNR and standard deviation across the split [S1, unofficial].
- Optional composite visualization of ground truth, rendered RGB, and depth [S2, unofficial].

**Data Structures:**
- `state_eval`: Evaluation state object containing inference variants of raymarching and rendering functions [S2, unofficial].
- `rendered_image`: Tuple or object containing `bg` (background color), `rgb` (rendered color), and `depth` [S2, unofficial].
- `gt_rgbs_f32`: Ground truth RGB images converted to float32 and composited with the rendered background [S2, unofficial].

**Steps:**
1. **Load Data:** Load ground truth images and camera poses for the specified split. If `half_res=True`, images are loaded at half resolution (e.g., 200x200 for Blender scenes) [S1, unofficial].
2. **Initialize Evaluation State:** Replace the training raymarching and rendering functions with their inference counterparts to optimize performance [S2, unofficial]:
   ```python
   state_eval = state.replace(raymarch=args.raymarch_eval).replace(render=args.render_eval)
   ```
3. **Iterate Views:** For each validation/test view:
   a. Extract camera transforms [S2, unofficial].
   b. Render the image using chunked rendering to manage memory [S1, unofficial]. The inference function returns `bg`, `rgb`, `depth`, and other metadata [S2, unofficial]:
      ```python
      bg, rgb, depth, _ = data.to_cpu(render_image_inference(KEY=key, transform_cw=val_transform, state=state_eval))
      ```
   c. Composite the ground truth RGBA image with the rendered background. This step is critical because NeRFs often render transparent backgrounds, while ground truth images may have alpha channels or require a white background assumption [S1, unofficial][S2, unofficial]:
      ```python
      # Convert GT RGBA to float32 [0, 1]
      gt_rgba_f32 = val_view.image_rgba_u8.astype(jnp.float32) / 255.0
      # Blend GT with rendered background
      gt_rgbs_f32 = data.blend_rgba_image_array(gt_rgba_f32, bg)
      ```
4. **Compute Metrics:**
   a. Calculate Mean Squared Error (MSE) between the composited ground truth RGB and the rendered RGB [S1, unofficial]:
      $$ \text{MSE} = \frac{1}{N} \sum_{i=1}^{N} (\text{pred}_i - \text{gt}_i)^2 $$
   b. Compute PSNR for the frame [S1, unofficial]:
      $$ \text{PSNR} = 10 \cdot \log_{10}\left(\frac{1}{\text{MSE}}\right) $$
      Note: Some implementations convert images to uint8 before comparison [S2, unofficial].
5. **Aggregate:** Accumulate PSNRs across all frames. Print per-frame PSNR and the mean ± standard deviation [S1, unofficial].
6. **Visualization (Optional):** Create side-by-side comparisons of ground truth, rendered RGB, and depth maps for qualitative inspection [S2, unofficial].

**Parameters:**
- `split`: Dataset split to evaluate (`"test"` or `"val"`) [S1, unofficial].
- `half_res`: Boolean flag to load images at half resolution (commonly used for Blender synthetic datasets) [S1, unofficial].

**Edge Cases and Pitfalls:**
- **Background Mismatch:** Failing to composite the ground truth with the rendered background (or vice versa) leads to incorrect PSNR if the NeRF renders transparency [S2, unofficial].
- **Resolution:** Using `half_res` speeds up evaluation but may underestimate quality for high-resolution scenes [S1, unofficial].
- **Chunking:** Rendering large images without chunking can cause out-of-memory errors [S1, unofficial].

## Operating characteristics
- **Time Complexity:** Linear with respect to the number of test views and pixels per view. Chunked rendering adds overhead but prevents memory spikes [S1, unofficial].
- **Memory Complexity:** Proportional to the image resolution and batch size of the chunked rendering [S1, unofficial].
- **Strengths:** Standard metric for NeRF quality; easy to implement; provides both quantitative (PSNR) and qualitative (visualizations) feedback [S1, unofficial][S2, unofficial].
- **Limitations:** PSNR is sensitive to pixel-wise errors and may not correlate perfectly with perceptual quality; does not account for geometric accuracy [S1, unofficial].
- **When to use:** During validation loops to monitor convergence and after training to report final model quality [S1, unofficial].

## Background
The PSNR evaluation pipeline is a standard component of Neural Radiance Field (NeRF) training and evaluation workflows, originating from the original NeRF paper and popularized by implementations like the NerfStudio and synthetic Blender dataset benchmarks [S1, unofficial]. It is widely used in academic research to compare different NeRF architectures and optimization strategies [S1, unofficial].

## Sources
[S1] yashtmy/nerf-3d-vision, https://github.com/yashtmy/nerf-3d-vision, fetched 2026-10-26 (unofficial)
[S2] blurgyy/jaxngp, https://github.com/blurgyy/jaxngp, fetched 2026-10-27 (unofficial)
