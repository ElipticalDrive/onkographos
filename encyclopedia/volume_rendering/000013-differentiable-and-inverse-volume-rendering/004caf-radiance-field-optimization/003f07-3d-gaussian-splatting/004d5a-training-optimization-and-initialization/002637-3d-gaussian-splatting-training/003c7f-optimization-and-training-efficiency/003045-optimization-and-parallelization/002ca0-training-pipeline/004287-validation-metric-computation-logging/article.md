# Validation Metric Computation & Logging

Also called: Testing Loop Evaluation, Validation Metrics

## Overview
This method computes quantitative validation metrics (primarily PSNR) and saves visual examples by rendering novel views from the optimized 3D Gaussian Splatting model and comparing them against ground truth images [S1, S2, unofficial]. It is distinct from the training loop as it operates in evaluation mode (`is_train=False`) with gradient tracking disabled, ensuring that the optimization state is not updated during metric calculation [S1, unofficial].

## How it works
**Inputs:**
- Optimized 3D Gaussian Splatting model parameters.
- Scene data containing training and testing camera poses.
- `testing_iterations`: A list of iteration indices at which validation is triggered [S2, unofficial].

**Outputs:**
- Mean PSNR score for test and training subsets.
- Saved JPEG images of rendered novel views for visual inspection [S1, unofficial].

**Data Structures:**
- `psnr_list`: A list of floating-point PSNR values collected during the validation pass.
- `validation_configs`: A dictionary mapping config names ('test', 'train') to lists of camera indices or objects [S2, unofficial].

**Steps:**
1. **Initialization**: Clear the CUDA cache to ensure no stale intermediate tensors remain. Initialize `psnr_list` as empty. Select a random index `show_idx` for visual output saving [S1, unofficial].
2. **Trigger Check**: At the current training iteration, check if it exists in `testing_iterations`. If not, skip validation [S2, unofficial].
3. **Configuration Setup**: Define `validation_configs`:
   - `'test'`: Use all cameras from `scene.getTestCameras()` [S2, unofficial].
   - `'train'`: Use 5 evenly spaced cameras from `scene.getTrainCameras()`, specifically indices 5, 10, 15, 20, 25, 30 (step 5) [S2, unofficial].
4. **Iteration Loop**: For each camera index in the selected configuration:
   a. Fetch data for the specific view. Ensure all tensors are moved to the GPU [S1, unofficial].
   b. Run the model forward pass with `is_train=False` and `torch.no_grad()` context [S1, unofficial].
   c. Render the image. Apply background color compositing if required [S1, unofficial].
   d. **Metric Computation**:
      - Clamp the rendered image and ground truth image to the range $[0.0, 1.0]$ [S2, unofficial].
      - Compute the Peak Signal-to-Noise Ratio (PSNR) between the rendered image $I_{render}$ and ground truth $I_{gt}$:
        $$ \text{PSNR} = 10 \cdot \log_{10} \left( \frac{1}{\text{MSE}(I_{render}, I_{gt})} \right) $$
        where $\text{MSE}$ is the mean squared error over all pixels [S2, unofficial].
      - (Optional) Compute L1 loss and SSIM if required by the specific implementation variant [S2, unofficial].
      - Append the PSNR value to `psnr_list` [S1, unofficial].
   e. **Visualization**: If the current index equals `show_idx`:
      - Detach the tensor from the computation graph.
      - Multiply pixel values by 255.
      - Permute dimensions from CHW to HWC.
      - Convert to NumPy array and save as a JPEG file [S1, unofficial].
5. **Aggregation**:
   - Compute the mean of `psnr_list`.
   - Round the result to 4 decimal places [S1, unofficial].
   - If the mean PSNR is less than 10, print an error message and exit the process [S1, unofficial].
   - Log the final PSNR value to the logger [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def compute_validation_metrics(model, scene, testing_iterations, current_iter, logger):
    # Step 1: Initialization
    torch.cuda.empty_cache()
    psnr_list = []
    show_idx = 0  # Arbitrary index for visualization
    
    # Step 2: Trigger Check
    if current_iter not in testing_iterations:
        return

    # Step 3: Configuration Setup
    test_cameras = scene.getTestCameras()
    train_cameras = scene.getTrainCameras()
    # Select 5 evenly spaced cameras from training set (indices 5 to 30 step 5)
    train_sample_indices = list(range(5, 31, 5))
    train_sample_cameras = [train_cameras[i] for i in train_sample_indices if i < len(train_cameras)]
    
    validation_configs = {
        'test': test_cameras,
        'train': train_sample_cameras
    }

    # Step 4: Iteration Loop
    for config_name, cameras in validation_configs.items():
        current_psnr_list = []
        
        for idx, camera in enumerate(cameras):
            # 4a: Fetch data
            data = fetch_data_for_camera(camera, phase='val')
            # Ensure data is on GPU
            for view in data.keys():
                for key in data[view].keys():
                    data[view][key] = data[view][key].cuda()
            
            # 4b: Forward pass
            with torch.no_grad():
                model.eval()
                rendered_data = model(data, is_train=False)
                # Apply background color if necessary
                rendered_img = rendered_data['novel_view']['img_pred'][0]
                rendered_img = pts2render(rendered_img, bg_color=0.0) # Simplified bg_color handling
            
            # 4c: Metric Computation
            gt_img = data['novel_view']['img_gt'][0]
            
            # Clamp to [0.0, 1.0]
            rendered_clamped = torch.clamp(rendered_img, 0.0, 1.0)
            gt_clamped = torch.clamp(gt_img, 0.0, 1.0)
            
            # Compute PSNR
            mse = torch.mean((rendered_clamped - gt_clamped) ** 2)
            if mse == 0:
                psnr_val = 100.0 # Avoid log(0)
            else:
                psnr_val = 10 * torch.log10(1.0 / mse)
            
            current_psnr_list.append(psnr_val.item())
            
            # 4d: Visualization
            if idx == show_idx:
                img_np = rendered_img.detach().cpu().numpy()
                img_np = (img_np * 255).astype(np.uint8)
                img_np = np.transpose(img_np, (1, 2, 0)) # HWC
                save_image(img_np, f"validation_{config_name}_{idx}.jpg")

        # Step 5: Aggregation for this config
        mean_psnr = sum(current_psnr_list) / len(current_psnr_list)
        mean_psnr = round(mean_psnr, 4)
        
        if mean_psnr < 10:
            logger.error(f"Validation PSNR too low: {mean_psnr}")
            exit()
            
        logger.info(f"{config_name} PSNR: {mean_psnr}")
```

**Parameters:**
- `testing_iterations`: List of integers specifying training iterations to run validation [S2, unofficial].
- `bg_color`: Background color used for compositing transparent renders (default often 0.0 or 1.0) [S1, unofficial].

**Edge Cases:**
- **MSE = 0**: If the rendered image is identical to the ground truth, PSNR is undefined (log(0)). Handle by assigning a high constant (e.g., 100.0) [G].
- **Empty Camera Lists**: If `getTestCameras()` returns an empty list, the loop should skip or raise an error [G].
- **Index Out of Bounds**: When selecting `train_sample_indices`, ensure indices do not exceed the length of `train_cameras` [S2, unofficial].

## Operating characteristics
- **Time Complexity**: $O(N \cdot T)$, where $N$ is the number of validation cameras and $T$ is the time to render a single view. Validation is only performed at specific iterations, not every step.
- **Memory Complexity**: $O(N)$ for storing PSNR values. CUDA cache is cleared before execution to prevent OOM errors during large scene validation [S1, unofficial].
- **Strengths**: Provides objective quantitative feedback (PSNR) and visual sanity checks (saved JPEGs) [S1, S2, unofficial].
- **Limitations**: Computationally expensive if `testing_iterations` are frequent. PSNR alone does not capture perceptual quality (SSIM/LPIPS may be needed) [S2, unofficial].

## Background
This procedure is part of the standard training pipeline for 3D Gaussian Splatting implementations, particularly those adapted for specific tasks like stereo or novel view synthesis where validation against held-out test sets is required [S1, S2, unofficial]. It builds on the general rasterization and optimization loop defined in [Training Pipeline](5325-training-pipeline.md).

## Sources
[S1] YaourtB/GPS_plus, https://github.com/YaourtB/GPS_plus, fetched 2026-10-27 (unofficial)
[S2] Yang-SyZng/AVM-GS, https://github.com/Yang-SyZng/AVM-GS, fetched 2026-10-27 (unofficial)
