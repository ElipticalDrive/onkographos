# Validation Metric Computation & Logging

Also called: Testing Loop Evaluation, Validation Metrics

## Overview
This method computes quantitative validation metrics (primarily PSNR) on novel views during training and logs detailed performance statistics to disk. It differs from the training loss computation by evaluating the model on held-out or sampled camera poses using full-resolution ground truth comparisons, rather than optimizing parameters via gradient descent. It builds on [3D Gaussian Splatting Training](3761-3d-gaussian-splatting-training.md) by providing the evaluation loop that runs at specified intervals.

## How it works
**Inputs:**
- Trained 3D Gaussian Splatting model (weights and scene representation).
- Scene object providing camera lists: `getTestCameras()` (full test set) and `getTrainCameras()` (training set).
- Configuration parameters: `testing_iterations` (list of iteration numbers to run validation), `show_idx` (index for saving a visual example), `show_path` (directory for saving JPEGs).

**Outputs:**
- Mean PSNR score (float, rounded to 4 decimal places).
- Optional saved JPEG image of a novel view.
- CSV log file with timing and resource usage.

**Data Structures:**
- `psnr_list`: List of floats storing PSNR values for each view.
- `validation_configs`: Dictionary mapping config names to camera lists.
  - `'test'`: List of all test cameras.
  - `'train'`: List of 5 evenly spaced training cameras (indices 5, 10, 15, 20, 25, 30 if available).

**Steps:**
1. **Initialization:**
   - Clear CUDA cache: `torch.cuda.empty_cache()` [S1, unofficial].
   - Initialize `psnr_list = []` [S1, unofficial].
   - Select `show_idx` (e.g., 0) for visualization [S1, unofficial].

2. **Trigger Check:**
   - If current iteration is not in `testing_iterations`, skip validation [S2, unofficial].

3. **Camera Selection:**
   - Define `validation_configs`:
     - `'test'`: `scene.getTestCameras()` [S2, unofficial].
     - `'train'`: 5 evenly spaced cameras from `scene.getTrainCameras()` [S2, unofficial].

4. **Evaluation Loop:**
   - For each camera in each config:
     - Fetch data for the camera [S1, unofficial].
     - Run model forward pass with `is_train=False` [S1, unofficial].
     - Render the view [S1, unofficial].
     - Clamp rendered image and ground truth to $[0.0, 1.0]$ [S2, unofficial].
     - Compute PSNR between rendered and ground truth images [S1, S2, unofficial].
     - Append PSNR to `psnr_list` [S1, unofficial].
     - If current index equals `show_idx`:
       - Detach tensor, multiply by 255, permute to HWC, convert to numpy [S1, unofficial].
       - Save as JPEG to `show_path` [S1, unofficial].

5. **Aggregation:**
   - Compute mean PSNR: $\text{mean\_psnr} = \frac{1}{N} \sum_{i=1}^{N} \text{PSNR}_i$ [S1, S2, unofficial].
   - Round to 4 decimal places [S1, unofficial].
   - If $\text{mean\_psnr} < 10$, print error and exit [S1, unofficial].
   - Log PSNR to logger [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def compute_validation_metrics(model, scene, iteration, testing_iterations, show_path, show_idx=0):
    if iteration not in testing_iterations:
        return None

    torch.cuda.empty_cache()
    psnr_list = []
    
    # Define camera sets
    test_cameras = scene.getTestCameras()
    train_cameras = scene.getTrainCameras()
    # Select 5 evenly spaced cameras from training set
    train_sample_indices = [5, 10, 15, 20, 25, 30]
    train_sample_indices = [i for i in train_sample_indices if i < len(train_cameras)]
    train_cameras_sampled = [train_cameras[i] for i in train_sample_indices]
    
    validation_configs = {
        'test': test_cameras,
        'train': train_sampled
    }
    
    for config_name, cameras in validation_configs.items():
        for idx, camera in enumerate(cameras):
            # Fetch data for the camera
            data = fetch_data(camera, phase='val')
            
            # Forward pass
            with torch.no_grad():
                data, _, _ = model(data, is_train=False)
                # Render novel view
                render_img = data['novel_view']['img_pred'][0]
                gt_img = data['novel_view']['img_gt'][0]
                
                # Clamp to [0.0, 1.0]
                render_img = torch.clamp(render_img, 0.0, 1.0)
                gt_img = torch.clamp(gt_img, 0.0, 1.0)
                
                # Compute PSNR
                psnr_value = compute_psnr(render_img, gt_img)
                psnr_list.append(psnr_value)
                
                # Save example image
                if idx == show_idx:
                    tmp_novel = render_img.detach() * 255
                    tmp_novel = tmp_novel.permute(1, 2, 0).cpu().numpy()
                    img_name = f"{show_path}/{config_name}_{idx}.jpg"
                    save_jpeg(tmp_novel, img_name)
    
    # Aggregate results
    mean_psnr = sum(psnr_list) / len(psnr_list)
    mean_psnr = round(mean_psnr, 4)
    
    if mean_psnr < 10:
        print(f"Error: PSNR {mean_psnr} < 10")
        exit()
        
    log_psnr(mean_psnr)
    return mean_psnr

def compute_psnr(render, gt):
    # PSNR = 10 * log10(1 / MSE)
    mse = torch.mean((render - gt) ** 2)
    if mse == 0:
        return float('inf')
    return 10 * torch.log10(1 / mse)

def save_jpeg(img_np, path):
    # Convert to uint8 and save
    img_uint8 = img_np.astype(np.uint8)
    cv2.imwrite(path, img_uint8[:, :, ::-1])
```

## Subtopics

### Training Performance Logging
Logs training performance metrics to a CSV file, including steps, timing, GPU samples, splats, VRAM, RSS, and pause state, with automatic session directory management [S3, unofficial]. It uses the environment variable `TRAIN_PERF` to enable logging: '1' logs to `run_dir`, other value logs to specified path, empty/disabled logs nothing [S3, unofficial]. The CSV header includes: `unix_ms,step,steps,interval_s,step_s,data_wait_s,gpu_sampled_s,gpu_sampled_steps,save_s,divisor,splats,vram_bytes,rss_bytes,paused` [S3, unofficial].

## Operating characteristics
- **Time Complexity:** Proportional to the number of validation cameras and the cost of a forward pass. Validation is typically run at sparse intervals (`testing_iterations`), so it does not dominate training time if intervals are large.
- **Memory Complexity:** Requires memory for rendering all validation views. `torch.cuda.empty_cache()` is called before validation to free unused memory [S1, unofficial].
- **Strengths:** Provides objective quality metrics (PSNR) and visual checks (JPEGs) to monitor convergence.
- **Limitations:** PSNR is not always correlated with perceptual quality. Computation can be slow for large test sets.
- **When to use:** During training to monitor model quality and detect overfitting or divergence.
- **When not to use:** For real-time inference or when computational resources are extremely limited.

## Background
Originated from the standard evaluation practices in 3D Gaussian Splatting implementations, where validation on novel views is essential for assessing reconstruction quality. The method is implemented in various open-source projects, including GPS_plus [S1, unofficial], AVM-GS [S2, unofficial], and spirula-studio [S3, unofficial].

## Sources
[S1] YaourtB/GPS_plus, https://github.com/YaourtB/GPS_plus, fetched 2026-10-27 (unofficial)
[S2] Yang-SyZng/AVM-GS, https://github.com/Yang-SyZng/AVM-GS, fetched 2026-10-27 (unofficial)
[S3] harry7557558/spirula-studio, https://github.com/harry7557558/spirula-studio, fetched 2026-10-27 (unofficial)
