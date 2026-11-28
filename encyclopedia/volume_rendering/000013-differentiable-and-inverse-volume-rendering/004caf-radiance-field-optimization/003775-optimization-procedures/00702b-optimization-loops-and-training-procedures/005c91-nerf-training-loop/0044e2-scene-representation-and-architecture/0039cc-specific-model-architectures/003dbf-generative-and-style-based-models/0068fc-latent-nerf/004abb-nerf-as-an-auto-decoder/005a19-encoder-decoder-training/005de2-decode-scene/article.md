# Decode-Scene


## Overview
Decode-Scene is a method for training neural radiance fields (NeRFs) in a latent space by supervising the pipeline with RGB images directly, rendering in the latent space and then decoding to RGB for the loss computation [S1, unofficial]. It differs from its sibling [Encode-Scene](17508-encode-scene.md) by computing the photometric loss in RGB space after decoding, rather than encoding target images into latent space first [S1, unofficial].

## How it works
**Inputs and Outputs:**
*   **Inputs:** Input RGB images $x_p$ with corresponding camera poses $p$, and a decoder network $D_\phi$ with trainable parameters $\phi$ that maps latent images to RGB space [S1, unofficial].
*   **Outputs:** Optimized latent scene representation parameters $\alpha$ and $T$ that minimize the decoding loss [S1, unofficial].

**Data Structures:**
*   **Latent NeRF ($R_\alpha$):** A neural radiance field that outputs a latent vector instead of RGB color [S1, unofficial].
*   **Decoder ($D_\phi$):** A neural network mapping rendered latent images to RGB space [S1, unofficial].

**Numbered Steps:**
1.  **Render Latent Image:** For a given camera pose $p$ and latent scene representation $(\alpha, T)$, render a latent image $L = R_\alpha(T, p)$ [S1, unofficial].
2.  **Decode to RGB:** Pass the rendered latent image $L$ through the decoder network to obtain the predicted RGB image: $\hat{x} = D_\phi(L)$ [S1, unofficial].
3.  **Compute Loss:** Calculate the photometric loss between the decoded RGB image $\hat{x}$ and the ground truth input image $x_p$ [S1, unofficial].

**Formula:**
The optimization objective is defined as:
$$ \min_{\alpha, T} L_D(\alpha, T) \triangleq \mathbb{E}_{x, p} \| x_p - D(R_\alpha(T, p)) \| $$
where:
*   $L_D(\alpha, T)$ is the Decode-Scene loss [S1, unofficial].
*   $x_p$ is the input RGB image for pose $p$ [S1, unofficial].
*   $D$ is the decoder network [S1, unofficial].
*   $R_\alpha(T, p)$ is the rendered latent image from the latent NeRF [S1, unofficial].
*   The norm $\| \cdot \|$ is generic in the source, typically representing $L_1$ or $L_2$ [S1, unofficial].

**Parameters:**
*   $\phi$: Trainable parameters of the decoder network $D_\phi$ [S1, unofficial].
*   $\alpha, T$: Parameters of the latent NeRF (e.g., Tri-Planes) [S1, unofficial].

**Pitfalls:**
*   **Performance:** This method is generally slower than [Encode-Scene](17508-encode-scene.md) because no latent images can be cached, and a gradient step requires differentiation through all parameters of the decoder for every training step [S1, unofficial].
*   **Precision:** Not given in the sources.

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def mse(a, b):
    """Mean squared error between two tensors."""
    return ((a - b) ** 2).mean()

def decode_scene_step(input_rgb, camera_pose, latent_nerf, decoder, alpha, T, phi):
    """One training step for Decode-Scene."""
    # Step 1: Render latent image from the latent NeRF
    latent_image = latent_nerf(alpha, T, camera_pose)
    # Step 2: Decode the latent image to RGB space
    decoded_rgb = decoder(latent_image, phi)
    # Step 3: Compute photometric loss in RGB space
    loss = mse(decoded_rgb, input_rgb)
    return loss
```

## Subtopics

### [Encode-Scene](17508-encode-scene.md)
Encode-Scene is a method for training 3D-aware latent scene representations by first encoding input RGB images into a lower-dimensional latent space and then optimizing a neural radiance field (NeRF) to render latent images that match these pre-computed encodings.

## Operating characteristics
*   **Time Complexity:** Slower than [Encode-Scene](17508-encode-scene.md) because the decoder must be evaluated and differentiated for every training step [S1, unofficial].
*   **Memory Complexity:** Not given in the sources.
*   **Strengths:** Supervises the pipeline with RGB images directly, ensuring high-fidelity color images in the output space [S1, unofficial].
*   **Limitations:** Computationally expensive due to the lack of caching and the need to backpropagate through the decoder [S1, unofficial].

## Background
Decode-Scene is a method for training NeRFs in a latent space described in [S1, unofficial]. It is part of a family of latent-space training methods that contrasts with [Encode-Scene](17508-encode-scene.md) and [Encode-Decode-Scene](17509-encode-decode-scene.md) [S1, unofficial].

## Sources
[S1] Exploring 3D-aware Latent Spaces for Efficiently Learning Numerous Scenes, https://doi.org/10.48550/arXiv.2403.11678, fetched 2026-10-29 (unofficial)
