# Spatiotemporal Volume Denoising with Background Fusion

Also called: Kernel prediction denoising with background fusion, Real-time volume-rendering image denoising based on kernel prediction

## Overview
This method denoises Monte Carlo volume rendering images by combining temporal reprojection with a learned kernel prediction network and fusing the result with the background using an alpha map. It addresses the specific challenge of preserving semi-transparent structures and sharp edges in volumetric data while maintaining real-time performance. Unlike direct prediction methods (e.g., FDAE) that rely solely on pixel-wise regression, this approach generates weighted combinations of neighboring samples, offering better structural preservation at the cost of struggling with extreme high-intensity outliers [S1, unofficial].

## How it works
The method operates on image-order rendering outputs. It takes the current noisy frame, the previous frame's denoised data, and the background color as inputs, producing a final denoised radiance image.

### Data Structures and Layout
*   **Input Images:** 1024x1024 resolution, 32-bit floating-point format [S1, unofficial].
*   **Alpha Map $A(i,j)$:** A single-channel map representing the transparency of the volume data at pixel coordinates $(i,j)$ [S1, unofficial].
*   **Background $B(i,j)$:** The color/value of the background at pixel coordinates $(i,j)$ [S1, unofficial].
*   **Training Patches:** 256x256 image patches extracted from the rendered images (four patches per image) [S1, unofficial].

### Numbered Steps
1.  **Reprojection:** Reproject the previous frame's denoised volume data into the current frame's coordinate space using camera motion and depth information [S1, unofficial].
2.  **Kernel Prediction:** Use a neural network (kernel prediction network) to predict spatial and temporal weight kernels based on the current noisy frame and the reprojected historical data [S1, unofficial].
3.  **Weighted Filtering:** Perform pixel-wise weighted filtering on the reprojected historical frame and the current noisy frame using the predicted kernels to obtain an intermediate denoised volume data $x_n'(i,j)$ [S1, unofficial].
4.  **Background Fusion:** Blend the intermediate denoised data $x_n'(i,j)$ with the background $B(i,j)$ using the alpha map $A(i,j)$ to produce the final denoised pixel value $x_n''(i,j)$ [S1, unofficial].
5.  **Failure Handling:** If the camera undergoes rapid motion, abrupt illumination changes, or if a ray passes entirely through transparent regions (where historical frame failure rate can reach 70%), copy the current frame's pixel value to replace the historical frame value, effectively degenerating to spatial denoising for that pixel [S1, unofficial].

### Formulas
**Background Fusion:**
The final denoised pixel value $x_n''$ is calculated as:
$$ x_n''(i,j) = A(i,j) \times x_n'(i,j) + (1 - A(i,j)) \times B(i,j) $$
where $A(i,j)$ is the alpha map and $B(i,j)$ is the background [S1, unofficial].

**Training Loss Functions:**
The model is trained using a weighted sum of gradient loss and Mean Squared Error (MSE) loss.

Gradient Loss $L_{grad}(I,R)$:
$$ L_{grad}(I,R) = \|\nabla_x I - \nabla_x R\|_2^2 + \|\nabla_y I - \nabla_y R\|_2^2 $$
where $I$ is the predicted image, $R$ is the reference image, and $\nabla_x, \nabla_y$ are the x and y gradients [S1, unofficial].

MSE Loss $L_{mse}(I,R)$:
$$ L_{mse}(I,R) = \|I - R\|_2^2 $$
where $\| \cdot \|_2^2$ denotes the squared L2 norm [S1, unofficial].

Total Loss $L_{total}(I,R)$:
$$ L_{total}(I,R) = w_{grad} \cdot L_{grad}(I,R) + w_{mse} \cdot L_{mse}(I,R) $$
where $w_{grad}$ and $w_{mse}$ are the weights for gradient and MSE losses, respectively [S1, unofficial].

**Evaluation Metrics:**
MSE:
$$ MSE(I,R) = \frac{1}{mn} \sum_{i=0}^{m-1} \sum_{j=0}^{n-1} [I(i,j) - R(i,j)]^2 $$
where $m,n$ are image dimensions [S1, unofficial].

PSNR:
$$ PSNR(I,R) = 10 \times \lg \frac{255^2}{MSE(I,R)} $$
where $\lg$ is the base-10 logarithm [S1, unofficial].

SSIM:
$$ SSIM(I,R) = \frac{(2\mu_I\mu_R+c_1)(\delta_{IR}+c_2)}{(\mu_I^2+\mu_R^2+c_1)(\delta_I^2+\delta_R^2+c_2)} $$
where $\mu_I, \mu_R$ are means, $\delta_I, \delta_R$ are standard deviations, and $\delta_{IR}$ is the covariance between images $I$ and $R$ [S1, unofficial].

Temporal PSNR (tPSNR):
$$ tPSNR(I,R) = PSNR(I_t - I_{t-1}, R_t - R_{t-1}) $$
where $t$ is the time index [S1, unofficial].

### Parameters
*   **$w_{grad}$:** Weight for gradient loss. Set to 0.4 to improve visual edge preservation [S1, unofficial].
*   **$w_{mse}$:** Weight for MSE loss. Set to 0.6 [S1, unofficial].
*   **Learning Rate:** 0.0001 (Adam optimizer) [S1, unofficial].
*   **Batch Size:** 4 [S1, unofficial].
*   **Training Epochs:** Converges after approximately 300 epochs [S1, unofficial].

### Edge Cases and Pitfalls
*   **High Gradient Weight:** If $w_{grad}$ is set too high (above 0.5), the image exhibits slight color distortion for some data [S1, unofficial].
*   **Transparent Regions:** In ray tracing, if a ray passes entirely through transparent regions, the historical frame failure rate can be as high as 70%. The method handles this by copying the current frame's pixel value to replace the historical value, degenerating to spatial denoising [S1, unofficial].
*   **Rapid Motion/Changes:** Historical frames become unusable if the camera undergoes rapid motion, illumination changes abruptly, or the transfer function varies quickly [S1, unofficial].
*   **Outliers:** The kernel prediction method struggles with highlighted outliers (e.g., channel values > 200 at 800 samples per pixel) that direct prediction methods can suppress [S1, unofficial].

### Pseudocode
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def spatiotemporal_denoising_with_background_fusion(
    current_noisy_frame: Image,
    previous_denoised_frame: Image,
    alpha_map: Image,
    background: Image,
    camera_transform: Matrix,
    depth_map: Image
) -> Image:
    """
    Denoises a volume rendering frame using spatiotemporal reprojection and background fusion.
    
    Args:
        current_noisy_frame: The current noisy rendered image.
        previous_denoised_frame: The denoised image from the previous frame.
        alpha_map: Transparency map for the volume data.
        background: Background color/image.
        camera_transform: Transformation matrix for reprojection.
        depth_map: Depth information for reprojection.
        
    Returns:
        The final denoised image.
    """
    
    # Step 1: Reprojection
    # Reproject previous denoised frame to current frame coordinates
    # This involves transforming pixels using camera_transform and depth_map
    reprojected_frame = reproject(previous_denoised_frame, camera_transform, depth_map)
    
    # Step 2: Kernel Prediction
    # Predict spatial and temporal weight kernels using a neural network
    # Inputs: current_noisy_frame, reprojected_frame
    # Output: weight_kernels (spatial and temporal weights)
    weight_kernels = predict_kernels(current_noisy_frame, reprojected_frame)
    
    # Step 3: Weighted Filtering
    # Apply predicted kernels to blend current and historical data
    intermediate_denoised = weighted_filter(
        current_noisy_frame, 
        reprojected_frame, 
        weight_kernels
    )
    
    # Step 4: Failure Handling (Optional/Implicit in some implementations)
    # Check for rapid motion or high transparency failure rates
    # If failure detected, replace intermediate_denoised with current_noisy_frame
    # For this implementation, we assume the kernel prediction handles 
    # or the user handles the fallback logic described in the source.
    # [G] Guard: Ensure reprojection validity before filtering.
    
    # Step 5: Background Fusion
    # Blend intermediate denoised data with background using alpha map
    final_denoised = alpha_map * intermediate_denoised + (1 - alpha_map) * background
    
    return final_denoised

def weighted_filter(current: Image, historical: Image, kernels: dict) -> Image:
    """
    Applies weighted filtering using predicted kernels.
    """
    # [G] Implementation detail: This function performs the pixel-wise 
    # weighted combination of neighboring samples as described in the source.
    # The exact kernel application depends on the specific network architecture 
    # (e.g., separable convolutions or explicit weighted sums).
    # Since the source describes it as "pixel-wise weighted filtering" 
    # on "historical frame reprojection result and the current frame result",
    # we assume a standard weighted sum operation.
    
    result = Image(current.width, current.height)
    for y in range(current.height):
        for x in range(current.width):
            # Get neighborhood samples from current and historical frames
            # Apply weights from kernels
            # Sum weighted samples
            result[y, x] = compute_weighted_sum(current, historical, x, y, kernels)
    return result

def compute_weighted_sum(current, historical, x, y, kernels):
    # [G] Placeholder for the specific kernel application logic.
    # The source states: "pixel-wise weighted filtering is performed on 
    # the historical frame reprojection result and the current frame result".
    # This implies a convolution-like operation or weighted average.
    # Without the exact network architecture details, we represent the 
    # conceptual operation.
    pass
```

## Subtopics

### Temporal Gradient Denoising
The parent method that applies denoising to gradient images using temporal reprojection. Spatiotemporal Volume Denoising with Background Fusion extends this by incorporating background fusion and kernel prediction for volume data. [Temporal Gradient Denoising](15080-temporal-gradient-denoising.md)

### Kernel Prediction Denoising
A class of methods that predict filtering kernels rather than direct pixel values. This method uses kernel prediction to produce outputs as weighted combinations of neighboring samples, distinguishing it from direct prediction methods like FDAE. [Kernel Prediction Denoising](52-kernel-prediction-denoising.md) (Hypothetical link based on context)

### Background Fusion
A technique to blend denoised volume data with the background using an alpha map, crucial for handling semi-transparent structures in volume rendering. [Background Fusion](52-background-fusion.md) (Hypothetical link based on context)

## Operating characteristics
*   **Time Complexity:** Inference time has an approximately quadratic relationship with input resolution [S1, unofficial].
*   **Memory Complexity:** Not given in the sources.
*   **Performance:** Prediction time is below 30 ms per frame across multiple test sets, comparable to WRLS, achieving real-time performance [S1, unofficial].
*   **Hardware:** Tested on NVIDIA GeForce RTX 4080 Ti GPU (12 GB VRAM) and Intel i7-12700KF CPU (32 GB RAM) [S1, unofficial].
*   **Strengths:** Preserves sharp features and reduces blurring; handles semi-transparent structures well; real-time performance.
*   **Limitations:** Struggles with highlighted outliers (e.g., channel values > 200); sensitive to rapid camera motion or illumination changes; historical frames can fail in transparent regions.
*   **When to Use:** Real-time volume rendering with semi-transparent structures where edge preservation is critical.
*   **When Not to Use:** Scenes with extreme high-intensity outliers or rapid, unpredictable camera/illumination changes.

## Background
The method was proposed in a study on real-time volume-rendering image denoising based on kernel prediction. It was developed to address the noise in Monte Carlo volume rendering while preserving the complex translucent structures (smoke, blood vessels, soft tissues) and dynamic shadows typical in volume data. The approach combines temporal coherence (reprojection) with a learned kernel prediction network and a final background fusion step. The training dataset consisted of 65 rendered image sequences from eight different volume data sets, including CT scans, with a resolution of 1024x1024. The model was trained using PyTorch on an NVIDIA RTX 4080 Ti GPU, converging after 300 epochs [S1, unofficial].

## Sources
[S1] Real-Time Volume-Rendering Image Denoising Based on Kernel Prediction, https://pmc.ncbi.nlm.nih.gov/articles/PMC12028196/, fetched 2026-10-26 (unofficial)
