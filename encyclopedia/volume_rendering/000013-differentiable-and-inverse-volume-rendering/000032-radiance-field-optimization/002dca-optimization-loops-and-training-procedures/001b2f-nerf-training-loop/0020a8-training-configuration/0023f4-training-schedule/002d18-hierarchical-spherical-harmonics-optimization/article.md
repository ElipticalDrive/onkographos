# Hierarchical Spherical Harmonics Optimization

Also called: Band-by-band optimization, SH Band Strategy

## Overview
Hierarchical Spherical Harmonics (SH) Optimization is a training schedule strategy for radiance-field optimization that iteratively refines the directional color representation of a scene by optimizing SH coefficients from a starting band up to a target maximum band [S1, unofficial]. It addresses the instability and slow convergence associated with initializing high-frequency SH bands simultaneously with low-frequency ones [S1, unofficial]. Unlike standard optimization which may initialize all bands at once, this method sequentially activates bands, allowing the scene density and low-frequency color to stabilize before introducing higher-frequency details [S1, unofficial].

## How it works
The method controls the optimization of Spherical Harmonics coefficients by managing which bands are active and how their parameters are initialized during the training loop. It relies on specific configuration flags to determine the starting band and initialization behavior.

### Inputs and Outputs
**Inputs:**
- `configs`: A configuration object containing:
  - `sphericalHarmonicsBand`: The maximum SH band index to optimize (integer).
  - `hierarchyBandStrategy`: Boolean flag. If `true`, bands are optimized hierarchically. If `false`, all bands may be optimized together depending on other flags.
  - `SHsForTargetResoOnly`: Boolean flag. If `true` and the current hierarchy level is not the minimum, the optimization resets to band 0.
  - `initialFromFile`: Boolean flag. If `true`, density initialization is skipped for specific band transitions.
  - `learningRate`: Base learning rate for scene parameters.
  - `shLearningRate`: Learning rate specifically for SH coefficients.
  - `epochs`: Total number of epochs per band stage.
  - `batchSize`: Number of rays per batch.
  - `solidStrategy`: Boolean flag. If `false`, self-sample offset is applied.
  - `varianceLossInterval`: Interval (in batches) for applying variance loss.
  - `varianceCoefficient`, `shCoefficient`: Weights for auxiliary losses.
  - `vairanceType`, `neighboorNums`: Parameters for variance loss calculation.
  - `maskLossCoefficient`, `huberDelta`: Parameters for RGB loss calculation.
  - `saveIntervals`: Interval (in epochs) for saving model checkpoints.
  - `justModel`: Boolean flag. If `true`, intermediate results are not saved.
- `this->scene`: The scene representation object, containing:
  - `optimizingBand`: The current SH band being optimized.
  - `stepSize`: The step size for ray marching.
  - `sparse`: Boolean flag indicating sparse representation.
  - `grayImage`: Boolean flag indicating if the scene is grayscale.
  - `setSelfSampleOffset(offset)`: Method to set sampling offset.
- `this->sceneOptimizer`: The optimizer object, containing:
  - `initialBand(band, densityInitialized)`: Initializes parameters for a specific band.
  - `initialAll()`: Initializes all parameters at once.
- `this->ray`: Ray data, containing:
  - `totalRays`: Total number of rays to process.
- `RenderUtils`: Rendering utilities, containing:
  - `renderRays(...)`: Renders rays for grayscale or basic loss.
  - `renderRays_withGradients_RGB(...)`: Renders rays with RGB gradients and loss computation.
  - `addVarianceLoss(...)`: Adds variance regularization loss.
- `Optimizer`: Optimizer utilities, containing:
  - `updateRGBScene(...)`: Updates RGB-specific scene parameters.
- `lossesDevice`, `lossesHost`: GPU and CPU arrays for storing per-ray losses.

**Outputs:**
- Updated scene parameters (density, SH coefficients, RGB).
- Loss metrics (min, max, average) per epoch.
- Saved model checkpoints at specified intervals.

### Data Structures
The implementation relies on the following data structures as defined in the source code:

1.  **Camera Structure** (`Camera`):
    -   `w2cExternalMat[16]`: World-to-camera transformation matrix (4x4).
    -   `intrinsicMat[9]`: Camera intrinsic matrix (3x3).
    -   `alpha[2]`, `principle[2]`: Focal length and principal point parameters.
    -   `translation[3]`: Camera center position.
    -   `rotationMat[9]`: Rotation matrix.
    -   Device pointers (`*c2wExternalMatDevice`, `*alphaDevice`, etc.) for GPU transfer.

2.  **Configuration Structure** (`configs`):
    -   Contains hyperparameters listed in Inputs.

3.  **Scene Structure** (`this->scene`):
    -   `optimizingBand`: Integer tracking the current SH band.
    -   `stepSize`: Float for ray marching step size.
    -   `sparse`: Boolean.
    -   `grayImage`: Boolean.

4.  **Loss Arrays**:
    -   `lossesDevice`: GPU array of `double` values, size `actualBatchSize`.
    -   `lossesHost`: CPU array of `double` values, size `actualBatchSize`.

### Numbered Steps

1.  **Determine Starting Band**:
    -   If `configs->hierarchyBandStrategy` is `false`, set `startBand = configs->sphericalHarmonicsBand`.
    -   Else, if `configs->SHsForTargetResoOnly` is `true` AND `currentHierarchy` is not `minimumHierarchy`, set `startBand = 0`.
    -   Else, `startBand` remains determined by the hierarchy strategy (typically 0 if hierarchy is active, or the max band if not). *Note: The source logic implies `startBand` is set based on these conditions before the loop.*

2.  **Iterate Through SH Bands**:
    -   Loop `optimizingBand` from `startBand` to `configs->sphericalHarmonicsBand` (inclusive).

3.  **Set Current Band**:
    -   Set `this->scene->optimizingBand = optimizingBand`.

4.  **Initialize Parameters**:
    -   Check condition: If `configs->hierarchyBandStrategy` is `true`, OR (`configs->hierarchyBandStrategy` is `false` AND `configs->SHsForTargetResoOnly` is `true` AND `currentHierarchy` is not `minimumHierarchy`):
        -   Determine `densityInitialized`:
            -   If `configs->initialFromFile` is `true` AND `optimizingBand == availableSHBand + 1`, then `densityInitialized = false`.
            -   Otherwise, `densityInitialized = true`.
        -   Call `this->sceneOptimizer.initialBand(optimizingBand, densityInitialized)`.
    -   Else (if `configs->hierarchyBandStrategy` is `false` AND NOT (`configs->SHsForTargetResoOnly` is `true` AND `currentHierarchy` is not `minimumHierarchy`)):
        -   Call `this->sceneOptimizer.initialAll()`.

5.  **Reset Learning Rates**:
    -   Set `configs->learningRate = rawLR` (initial raw learning rate).
    -   Set `configs->shLearningRate = rawSHLR` (initial raw SH learning rate).

6.  **Epoch Loop**:
    -   Loop `epoch` from 0 to `configs->epochs` (inclusive).

7.  **Batch Calculation**:
    -   Calculate `batchCount`:
        $$ \text{batchCount} = \lceil \frac{\text{this->ray.totalRays}}{\text{configs->batchSize}} \rceil = \frac{\text{this->ray.totalRays} + \text{configs->batchSize} - 1}{\text{configs->batchSize}} $$
        *(Integer division rounding up)*.

8.  **Batch Loop**:
    -   Loop `batchIndex` from 0 to `batchCount - 1`.

9.  **Sample Index and Batch Size**:
    -   Calculate `startSampleIndex = batchIndex * configs->batchSize`.
    -   Calculate `actualBatchSize`:
        -   If `startSampleIndex + configs->batchSize < this->ray.totalRays`, then `actualBatchSize = configs->batchSize`.
        -   Else, `actualBatchSize = this->ray.totalRays - startSampleIndex`.

10. **Set Sampling Offset**:
    -   If `!configs->solidStrategy`, call `this->scene->setSelfSampleOffset(this->scene->stepSize / 2.0)`.

11. **Render Rays (Grayscale/Basic)**:
    -   Call `RenderUtils::renderRays(ray, scene, sampleRaysIndexGPU, startSampleIndex, actualBatchSize)`.

12. **Render Rays (RGB/Gradients)**:
    -   If `renderRes.grayImage` is `false`:
        -   Call `RenderUtils::renderRays_withGradients_RGB(sceneOptimizer, ray, scene, lossesDevice, sampleRaysIndexGPU, startSampleIndex, actualBatchSize, configs->maskLossCoefficient, configs->huberDelta)`.

13. **Copy Losses to Host**:
    -   Copy `actualBatchSize` doubles from `lossesDevice` to `lossesHost`:
        ```python
        cudaMemcpy(lossesHost, lossesDevice, sizeof(double) * actualBatchSize, cudaMemcpyDeviceToHost)
        ```

14. **Compute Loss**:
    -   Calculate `loss = getMSEError(lossesHost, actualBatchSize)`.
    -   If `isnan(loss)`:
        -   Print error.
        -   Free memory.
        -   Return (terminate training).

15. **Update Loss Statistics**:
    -   If `loss > maxLoss`, set `maxLoss = loss`.
    -   If `minLoss > loss` (or if `minLoss` is uninitialized), set `minLoss = loss`.
    -   Accumulate `losses += loss`.

16. **Add Variance Loss**:
    -   If `!scene->sparse` OR `batchIndex % configs->varianceLossInterval == 0`:
        -   Call `RenderUtils::addVarianceLoss(scene, sceneOptimizer, configs->varianceCoefficient, configs->shCoefficient, configs->vairanceType, configs->neighboorNums)`.

17. **Update RGB Scene**:
    -   If `scene->grayImage` is `false`:
        -   Call `Optimizer::updateRGBScene(sceneOptimizer, scene, timeStep)`.

18. **Decay Learning Rates**:
    -   Update `configs->learningRate`:
        $$ \text{configs->learningRate} = \frac{\text{configs->learningRate}}{2^{(1.0 / \text{batchCount}) / (\text{epoch} + 1.0)}} $$
    -   Update `configs->shLearningRate`:
        $$ \text{configs->shLearningRate} = \frac{\text{configs->shLearningRate}}{2^{(1.0 / \text{batchCount}) / (\text{epoch} + 1.0)}} $$

19. **Store Epoch Losses**:
    -   After the batch loop, store metrics in `lossesPerEpochs` array at indices `epoch*4` to `epoch*4+3`:
        -   `lossesPerEpochs[epoch*4+0] = minLoss`
        -   `lossesPerEpochs[epoch*4+1] = maxLoss`
        -   `lossesPerEpochs[epoch*4+2] = losses / batchCount` (Average loss)
        -   `lossesPerEpochs[epoch*4+3] = GPU elapsed time`

20. **Save Model**:
    -   If `!configs->justModel` AND `(epoch + 1) % configs->saveIntervals == 0`:
        -   Save experiment results to directory named `"Band_%d"` (where `%d` is the current `optimizingBand`).

### Pseudocode

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def hierarchical_sh_optimization(configs, scene, scene_optimizer, ray, render_utils, optimizer):
    """
    Implements Hierarchical Spherical Harmonics Optimization.
    """
    # 1. Determine Starting Band
    if not configs.hierarchyBandStrategy:
        start_band = configs.spherical_harmonics_band
    else:
        if configs.shs_for_target_reso_only and current_hierarchy != minimum_hierarchy:
            start_band = 0
        else:
            # Default behavior for hierarchy strategy or when SHsForTargetResoOnly is false
            # Source implies start_band is set, but exact default for hierarchyBandStrategy=true
            # when SHsForTargetResoOnly is false is not explicitly stated as a number,
            # but typically starts from 0 in hierarchical approaches.
            # Based on "iterating ... from a starting band up to a target band",
            # and the condition logic, we assume 0 if not overridden.
            start_band = 0 

    # 2. Iterate Through SH Bands
    for optimizing_band in range(start_band, configs.spherical_harmonics_band + 1):
        scene.optimizing_band = optimizing_band
        
        # 4. Initialize Parameters
        init_condition = (configs.hierarchyBandStrategy) or \
                         (not configs.hierarchyBandStrategy and configs.shs_for_target_reso_only and current_hierarchy != minimum_hierarchy)
        
        if init_condition:
            if configs.initial_from_file and optimizing_band == available_sh_band + 1:
                density_initialized = False
            else:
                density_initialized = True
            scene_optimizer.initial_band(optimizing_band, density_initialized)
        else:
            scene_optimizer.initial_all()
            
        # 5. Reset Learning Rates
        configs.learning_rate = raw_lr
        configs.sh_learning_rate = raw_sh_lr
        
        # 6. Epoch Loop
        for epoch in range(configs.epochs + 1):
            # 7. Batch Calculation
            batch_count = (ray.total_rays + configs.batch_size - 1) // configs.batch_size
            
            max_loss = float('-inf')
            min_loss = float('inf')
            total_loss = 0.0
            
            # 8. Batch Loop
            for batch_index in range(batch_count):
                start_sample_index = batch_index * configs.batch_size
                
                # 9. Sample Index and Batch Size
                if start_sample_index + configs.batch_size < ray.total_rays:
                    actual_batch_size = configs.batch_size
                else:
                    actual_batch_size = ray.total_rays - start_sample_index
                
                # 10. Set Sampling Offset
                if not configs.solid_strategy:
                    scene.set_self_sample_offset(scene.step_size / 2.0)
                
                # 11. Render Rays (Grayscale/Basic)
                render_utils.render_rays(ray, scene, sample_rays_index_gpu, start_sample_index, actual_batch_size)
                
                # 12. Render Rays (RGB/Gradients)
                if not render_res.gray_image:
                    render_utils.render_rays_with_gradients_rgb(
                        scene_optimizer, ray, scene, losses_device, 
                        sample_rays_index_gpu, start_sample_index, actual_batch_size,
                        configs.mask_loss_coefficient, configs.huber_delta
                    )
                
                # 13. Copy Losses to Host
                # cudaMemcpy(lossesHost, lossesDevice, sizeof(double) * actualBatchSize, cudaMemcpyDeviceToHost)
                losses_host = copy_device_to_host(losses_device, actual_batch_size)
                
                # 14. Compute Loss
                loss = get_mse_error(losses_host, actual_batch_size)
                if isnan(loss):
                    print("Error: Loss is NaN")
                    free_memory()
                    return
                
                # 15. Update Loss Statistics
                if loss > max_loss:
                    max_loss = loss
                if loss < min_loss:
                    min_loss = loss
                total_loss += loss
                
                # 16. Add Variance Loss
                if not scene.sparse or batch_index % configs.variance_loss_interval == 0:
                    render_utils.add_variance_loss(
                        scene, scene_optimizer, configs.variance_coefficient,
                        configs.sh_coefficient, configs.variance_type, configs.neighboor_nums
                    )
                
                # 17. Update RGB Scene
                if not scene.gray_image:
                    optimizer.update_rgb_scene(scene_optimizer, scene, time_step)
                
                # 18. Decay Learning Rates
                decay_factor = 2.0 ** ((1.0 / batch_count) / (epoch + 1.0))
                configs.learning_rate /= decay_factor
                configs.sh_learning_rate /= decay_factor
            
            # 19. Store Epoch Losses
            losses_per_epochs[epoch * 4 + 0] = min_loss
            losses_per_epochs[epoch * 4 + 1] = max_loss
            losses_per_epochs[epoch * 4 + 2] = total_loss / batch_count
            losses_per_epochs[epoch * 4 + 3] = gpu_elapsed_time
            
            # 20. Save Model
            if not configs.just_model and (epoch + 1) % configs.save_intervals == 0:
                save_experiment_results(directory_name=f"Band_{optimizing_band}")
```

### Parameters
-   `configs->sphericalHarmonicsBand`: Maximum SH band index. Typical values depend on desired detail (e.g., 3 or 4 for NeRF-like scenes). Not given in the sources.
-   `configs->hierarchyBandStrategy`: Boolean. Controls hierarchical vs. flat optimization.
-   `configs->SHsForTargetResoOnly`: Boolean. Resets start band to 0 if true and not at minimum hierarchy.
-   `configs->initialFromFile`: Boolean. Skips density re-initialization if true and band matches `availableSHBand+1`.
-   `configs->learningRate`, `configs->shLearningRate`: Base learning rates. Reset at each band start.
-   `configs->epochs`: Number of epochs per band.
-   `configs->batchSize`: Batch size for ray sampling.
-   `configs->solidStrategy`: If false, applies self-sample offset.
-   `configs->varianceLossInterval`: Interval for variance loss application.
-   `configs->varianceCoefficient`, `configs->shCoefficient`: Weights for variance loss.
-   `configs->vairanceType`, `configs->neighboorNums`: Parameters for variance loss.
-   `configs->maskLossCoefficient`, `configs->huberDelta`: Parameters for RGB loss.
-   `configs->saveIntervals`: Interval for saving models.
-   `configs->justModel`: If true, skips saving intermediate results.

### Edge Cases and Pitfalls
-   **NaN Loss**: If the computed MSE loss is NaN, the training terminates immediately, freeing memory [S1, unofficial].
-   **Boundary Conditions**: The last batch may have fewer rays than `batchSize`. The code explicitly calculates `actualBatchSize` to handle this [S1, unofficial].
-   **Density Initialization**: The `densityInitialized` flag depends on `initialFromFile` and the relationship between `optimizingBand` and `availableSHBand`. If `optimizingBand == availableSHBand + 1` and `initialFromFile` is true, density is not re-initialized [S1, unofficial].
-   **Learning Rate Decay**: The decay factor is applied per batch, not per epoch, which can lead to significant decay over many batches [S1, unofficial].

## Operating characteristics
-   **Time Complexity**: The method adds overhead by iterating through SH bands. For each band, it runs a full training loop (epochs * batches). If the maximum band is $B$, and each band requires $E$ epochs and $N$ batches, the total complexity is roughly $O(B \cdot E \cdot N)$. However, since lower bands are simpler, the per-band cost may decrease, but the source does not provide specific timing data.
-   **Memory Complexity**: Memory usage is dominated by the scene representation and the SH coefficients. Optimizing bands sequentially allows for potential memory savings if lower bands are discarded, but the source does not specify memory management details beyond the standard scene and loss arrays.
-   **Performance**: The method aims to improve convergence stability by avoiding the initialization of high-frequency bands too early. This can lead to better final quality but increases total training time due to the sequential nature.
-   **Strengths**: Prevents instability from high-frequency SH coefficients; allows for controlled refinement of detail.
-   **Limitations**: Increases training time; requires careful tuning of band transition parameters (`hierarchyBandStrategy`, `SHsForTargetResoOnly`).
-   **When to Use**: When training radiance fields with SH representations and experiencing instability or poor convergence with high-frequency details.
-   **When Not to Use**: When training time is critical and the scene does not require high-frequency detail; or when using a representation that does not support SH bands.

## Background
Hierarchical Spherical Harmonics Optimization is a training schedule strategy used in differentiable volume rendering, specifically in the context of NeRF (Neural Radiance Fields) and related radiance-field optimizations. It is implemented in the `Clear-plenoxels` repository [S1, unofficial]. The method addresses the challenge of optimizing high-dimensional SH coefficients by breaking the problem into smaller, more manageable sub-problems (bands). This approach is similar to other hierarchical optimization techniques in graphics, such as multi-resolution rendering or progressive refinement.

## Sources
[S1] nortonBryan/Clear-plenoxels, https://github.com/nortonBryan/Clear-plenoxels, fetched 2026-10-27 (unofficial)
