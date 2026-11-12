# Adaptive Volume Renderer (AVR)
Also called: Adapted version of differentiable ray marching algorithm, RM-LSTM based adaptive sampling

## Overview
The Adaptive Volume Renderer (AVR) is a hybrid rendering method that combines adaptive ray marching with standard volume rendering to reduce computational cost while maintaining visual fidelity [S1, unofficial]. It addresses the high latency of standard volume rendering in neural radiance fields (NeRFs) by using a recurrent neural network to predict a surface intersection point, then performing localized volume rendering around that point [S1, unofficial]. Unlike pure ray marching methods that output a single color from one sample, AVR samples multiple points around the predicted depth to recover fine details, achieving quality comparable to standard volume rendering but significantly faster [S1, unofficial].

## How it works
**Inputs and Outputs:**
*   **Input:** A set of source images and their corresponding camera poses.
*   **Output:** A rendered RGB image for a target view.
*   **Internal State:** A CNN encoder (e.g., ResNet-34) extracts latent features from source images. An RM-LSTM processes these features to predict depth.

**Data Structures:**
*   **RM-LSTM:** A Long Short-Term Memory network with 512 feature channels [S1, unofficial].
*   **Ray Parameters:** Depth $d$ is updated iteratively. The ray is defined by origin and direction, but the method operates primarily on the depth parameter along the ray [S1, unofficial].

**Algorithm Steps:**
1.  **Feature Extraction:** Extract latent features $F$ from the source images using a pretrained CNN encoder (e.g., ResNet-34) [S1, unofficial].
2.  **Depth Prediction (RM-LSTM):**
    *   Initialize the LSTM state.
    *   Feed features $F$ into the RM-LSTM.
    *   The RM-LSTM runs for $K=10$ steps [S1, unofficial].
    *   At each step, the LSTM predicts a step length $\Delta d$ and updates the current depth $d$.
    *   The final output is the predicted depth $d_{pred}$ [S1, unofficial].
3.  **Regularization (Training Only):** During training, penalize the final depth $d_{pred}$ if it falls outside the bounds $[d_{near}, d_{far}] = [0.5, 2.0]$ [S1, unofficial].
4.  **Local Sampling:** Sample $n=10$ points uniformly within a neighborhood of radius $\epsilon = 0.05$ around $d_{pred}$ [S1, unofficial]. The sampled depths are $d_i = d_{pred} + \delta_i$, where $\delta_i \in [-0.05, 0.05]$ [S1, unofficial].
5.  **Volume Rendering:**
    *   For each sampled point $d_i$, query the neural radiance field (NeRF) to obtain color $c_i$ and density $\sigma_i$ [S1, unofficial].
    *   Perform standard volume rendering integration along these points to calculate the final RGB value [S1, unofficial].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def adaptive_volume_rendering(source_images, camera_poses, target_pose):
    # 1. Feature Extraction
    # Uses a pretrained ResNet-34 encoder from PixelNeRF
    latent_features = cnn_encoder(source_images)
    
    # 2. RM-LSTM Depth Prediction
    # RM-LSTM has 512 feature channels and runs for 10 steps
    lstm = RMLSTM(feature_channels=512, num_steps=10)
    predicted_depth = lstm.predict(latent_features)
    
    # 3. Local Sampling
    # Sample n=10 points around +/-0.05 neighborhood of predicted depth
    num_samples = 10
    neighborhood_radius = 0.05
    sample_offsets = uniform_random(num_samples, min=-neighborhood_radius, max=neighborhood_radius)
    sampled_depths = predicted_depth + sample_offsets
    
    # 4. Volume Rendering
    # Query NeRF for color and density at sampled depths
    colors = []
    densities = []
    for depth in sampled_depths:
        c, sigma = nerf_query(target_pose, depth)
        colors.append(c)
        densities.append(sigma)
        
    # Perform standard volume rendering integration
    # Note: Standard volume rendering formula requires proper ordering and alpha compositing
    # Assuming sampled_depths are sorted or processed in ray order
    final_rgb = standard_volume_rendering_integration(colors, densities)
    
    return final_rgb
```

**Parameters:**
*   RM-LSTM feature channels: 512 [S1, unofficial].
*   RM-LSTM steps: 10 [S1, unofficial].
*   Volume rendering sample count ($n$): 10 [S1, unofficial].
*   Sampling neighborhood radius: $\pm 0.05$ [S1, unofficial].
*   Depth bounds (regularization): Near plane 0.5, Far plane 2.0 [S1, unofficial].

**Edge Cases and Pitfalls:**
*   **Depth Bounds:** If the predicted depth falls outside $[0.5, 2.0]$, the model may produce artifacts or incorrect colors. The regularization term helps mitigate this during training [S1, unofficial].
*   **Neighborhood Size:** The fixed radius of 0.05 assumes a certain scale of the scene. If the scene geometry varies significantly, this fixed neighborhood may be too small or too large, affecting quality [S1, unofficial].
*   **Computational Constraints:** The number of LSTM steps is limited to 10 due to computational constraints, which may limit the accuracy of the depth prediction [S1, unofficial].

## Operating characteristics
*   **Performance:** On the ShapeNet cars dataset, AVR achieves a PSNR of 23.22, SSIM of 0.9056, and LPIPS of 0.108 [S1, unofficial].
*   **Speed:** AVR renders at 4.2 fps, which is approximately 7 times faster than standard volume rendering (0.4 fps) [S1, unofficial].
*   **Comparison to Raymarcher:** AVR has slightly better quality than the Raymarcher (PSNR 23.22 vs 23.02) but is significantly slower (4.2 fps vs 34.9 fps) [S1, unofficial].
*   **Comparison to Standard Volume Rendering:** AVR achieves comparable quality to standard volume rendering (PSNR 23.34) with a significant speedup [S1, unofficial].
*   **Strengths:** Balances speed and quality better than pure ray marching; leverages learned features for efficient depth prediction.
*   **Limitations:** Requires training a neural network (RM-LSTM) alongside the NeRF; performance is dependent on the accuracy of the depth prediction.

## Background
The Adaptive Volume Renderer was proposed by Philipp Henzler, Jeremy Reizenstein, Patrick Labatut, Roman Shapovalov, and Tobias as an extension of the PixelNeRF architecture [S1, unofficial]. It was developed to address the computational cost of standard volume rendering in neural radiance fields by integrating adaptive ray marching techniques [S1, unofficial]. The method builds on the idea that predicting the surface location can significantly reduce the number of samples needed for accurate rendering [S1, unofficial].

## Sources
[S1] Combining Volume Rendering with Adaptive Ray Marching in Neural ..., https://adaptivevolumerenderer.github.io/, fetched 2026-10-28 (unofficial)
