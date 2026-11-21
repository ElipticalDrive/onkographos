# ENTIRE

Also called: Learning-based Volume Rendering Time Prediction, rEnderiNg TIme pREdiction network

## Overview
ENTIRE is a deep learning-based method for predicting volume rendering time that decouples volume feature extraction from time prediction to handle multiple factors including volume data characteristics, image resolution, camera configuration, and transfer function settings [S1, unofficial]. It consists of two stages: an autoencoder (VolumeNet) extracts a compact feature vector from the volume, and a prediction network (PredNet) estimates the rendering time using this vector along with rendering parameters [S1, unofficial]. Unlike its sibling [Low Volume Resolution-based Prediction](17890-low-volume-resolution-based-prediction.md), which relies on rendering at lower resolution and linear scaling, ENTIRE uses a neural network to capture non-linear effects and generalizes across different hardware and rendering methods without manual adjustment [S1, unofficial].

## How it works
**Inputs and Outputs:**
*   **Input:** A 3D volume $V_i$ (downsampled to a fixed resolution, typically $128^3$), image resolution, camera pose $\eta$, and transfer function parameters $\kappa$.
*   **Output:** Predicted rendering time $t_{pred}$.

**Data Structures:**
1.  **VolumeNet (Autoencoder):**
    *   **Encoder:** 5 convolutional layers followed by a fully connected layer.
    *   **Decoder:** Transposed convolutional layers.
    *   **Feature Vector:** Output of the encoder, dimension $F$ (typically 2–4).
2.  **PredNet (Prediction Network):**
    *   Fully connected layers.
    *   Input: Concatenation of Feature Vector, Camera Pose, Transfer Function, and Image Resolution.
    *   Output: Scalar $t_{pred}$.

**Parameters:**
*   **Volume Resolution:** Input to VolumeNet is downsampled to $128^3$ (or $64^3$ for specific configurations) [S1, unofficial].
*   **Feature Vector Dimension ($F$):** Output dimension of VolumeNet. Best performance observed at $F=2$ or $F=4$ for CUDA raycaster, $F=32$ for yt renderer [S1, unofficial].
*   **Transfer Function Representation ($\kappa$):** Parameter vector $\kappa = (c_0, w_0, h_0, ..., c_{m-1}, w_{m-1}, h_{m-1})$ representing Gaussian lobes [S1, unofficial].
*   **Camera Pose ($\eta$):** Vector consisting of rotation angles $R = (R_x, R_y)$ in degrees and translation distance $D_z$ along the z-axis [S1, unofficial].
*   **Optimizer:** Adam with $\beta_1 = 0.9, \beta_2 = 0.999$ [S1, unofficial].
*   **Learning Rate:** VolumeNet: $[10^{-3}, 10^{-5}]$; PredNet: $[10^{-4}, 10^{-6}]$ [S1, unofficial].
*   **Batch Size:** 16 [S1, unofficial].
*   **Epochs:** 200 for both networks [S1, unofficial].
*   **Gradient Clipping:** Max norm 1.0 [S1, unofficial].

**Steps:**
1.  **Downsample:** Downsample original volumes to a fixed resolution (e.g., $128^3$) for VolumeNet input.
2.  **Train VolumeNet:** Train the autoencoder to reconstruct the volume $V_i$ and output feature vector $X_i$.
    *   Loss: Reconstruction loss $\mathcal{L}_{rec}$.
    *   Formula: $X_i = f_{FC} \left( \frac{1}{K} \left( \bigoplus_{u=1...5} \text{ReLU}(I_{\text{conv}}) \right) \right)$ [S1, unofficial].
    *   Note: High-fidelity reconstruction is not required; the vector must be distinct and expressive [S1, unofficial].
3.  **Capture Ground Truth:** Render the original full-resolution volumes with various camera poses, transfer functions, and resolutions to capture ground-truth rendering times $t$.
4.  **Train PredNet:** Train the prediction network using the feature vectors $X_i$, rendering parameters, and captured times $t$.
    *   Loss: Prediction loss $\mathcal{L}_{pred} = \frac{1}{2K} \sum_{k=0}^{K-1} (t_k - t_k^{pred})^2$ [S1, unofficial].
    *   Formula: $t_i^{pred} = \psi(X_i, \eta, \kappa)$ [S1, unofficial].
5.  **Inference:**
    *   Run VolumeNet once per volume to get $X_i$.
    *   Run PredNet with $X_i$, current $\eta$, $\kappa$, and resolution to get $t_{pred}$.

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import torch
import torch.nn as nn
import torch.optim as optim

class VolumeNet(nn.Module):
    def __init__(self, feature_dim=4):
        super(VolumeNet, self).__init__()
        # Encoder: 5 Conv layers + FC
        self.encoder = nn.Sequential(
            nn.Conv3d(1, 32, kernel_size=3, stride=1, padding=1),
            nn.ReLU(),
            nn.Conv3d(32, 64, kernel_size=3, stride=1, padding=1),
            nn.ReLU(),
            nn.Conv3d(64, 128, kernel_size=3, stride=1, padding=1),
            nn.ReLU(),
            nn.Conv3d(128, 128, kernel_size=3, stride=1, padding=1),
            nn.ReLU(),
            nn.Conv3d(128, 128, kernel_size=3, stride=1, padding=1),
            nn.ReLU(),
            nn.AdaptiveAvgPool3d(1), # Global average pooling to flatten
            nn.Flatten(),
            nn.Linear(128, feature_dim)
        )
        # Decoder: Transposed convolutions for reconstruction loss
        self.decoder = nn.Sequential(
            nn.Linear(feature_dim, 128),
            nn.ReLU(),
            nn.Unflatten(1, (128, 1, 1, 1)),
            nn.ConvTranspose3d(128, 128, kernel_size=3, stride=1, padding=1),
            nn.ReLU(),
            nn.ConvTranspose3d(128, 128, kernel_size=3, stride=1, padding=1),
            nn.ReLU(),
            nn.ConvTranspose3d(128, 64, kernel_size=3, stride=1, padding=1),
            nn.ReLU(),
            nn.ConvTranspose3d(64, 32, kernel_size=3, stride=1, padding=1),
            nn.ReLU(),
            nn.ConvTranspose3d(32, 1, kernel_size=3, stride=1, padding=1)
        )

    def forward(self, x):
        # x shape: (batch, 1, D, H, W)
        feature = self.encoder(x)
        recon = self.decoder(feature)
        return feature, recon

class PredNet(nn.Module):
    def __init__(self, feature_dim=4, cam_dim=3, tf_dim=3*m, res_dim=2):
        super(PredNet, self).__init__()
        input_dim = feature_dim + cam_dim + tf_dim + res_dim
        self.network = nn.Sequential(
            nn.Linear(input_dim, 256),
            nn.SELU(), # Using SELU as per source description
            nn.Linear(256, 256),
            nn.SELU(),
            nn.Linear(256, 1)
        )

    def forward(self, feature, cam_pose, tf_params, resolution):
        # Concatenate inputs
        # feature: (batch, feature_dim)
        # cam_pose: (batch, cam_dim)
        # tf_params: (batch, tf_dim)
        # resolution: (batch, res_dim)
        combined = torch.cat([feature, cam_pose, tf_params, resolution], dim=1)
        pred_time = self.network(combined)
        return pred_time

def train_entire(volumes, ground_truth_times, camera_poses, tf_params, resolutions):
    # 1. Initialize models
    volume_net = VolumeNet(feature_dim=4)
    pred_net = PredNet(feature_dim=4, cam_dim=3, tf_dim=9, res_dim=2) # Example dims
    
    # 2. Optimizers
    vol_opt = optim.Adam(volume_net.parameters(), lr=1e-3)
    pred_opt = optim.Adam(pred_net.parameters(), lr=1e-4)
    
    # 3. Training Loop
    for epoch in range(200):
        # Stage 1: Train VolumeNet
        volume_net.train()
        vol_opt.zero_grad()
        features, reconstructions = volume_net(volumes)
        rec_loss = nn.MSELoss()(reconstructions, volumes)
        rec_loss.backward()
        torch.nn.utils.clip_grad_norm_(volume_net.parameters(), max_norm=1.0)
        vol_opt.step()
        
        # Stage 2: Train PredNet
        pred_net.train()
        pred_opt.zero_grad()
        # Note: In practice, VolumeNet weights might be frozen or trained jointly
        # Here we assume features are extracted
        pred_times = pred_net(features, camera_poses, tf_params, resolutions)
        pred_loss = nn.MSELoss()(pred_times.squeeze(), ground_truth_times)
        pred_loss.backward()
        torch.nn.utils.clip_grad_norm_(pred_net.parameters(), max_norm=1.0)
        pred_opt.step()
        
        # Early stopping check on validation loss would occur here

def predict_time(volume, camera_pose, tf_params, resolution, volume_net, pred_net):
    # 1. Downsample volume if necessary (handled by input preprocessing)
    # 2. Extract feature
    volume_net.eval()
    with torch.no_grad():
        feature, _ = volume_net(volume)
    # 3. Predict time
    pred_net.eval()
    with torch.no_grad():
        t_pred = pred_net(feature, camera_pose, tf_params, resolution)
    return t_pred
```

**Edge Cases and Pitfalls:**
*   **Systematic Bias:** ENTIRE tends to underpredict when ground-truth rendering times are large ($\approx 0.15$s) due to training data imbalance where high-cost scenarios comprise $< 10\%$ of samples [S1, unofficial].
*   **Feature Dimension:** Setting $F=1$ is insufficient for complex structural variations; $F=2$ is the minimum viable dimension. Excessively high $F$ introduces noise and diminishes generalization [S1, unofficial].
*   **Resolution Impact:** Higher input resolution for VolumeNet improves reconstruction but does not linearly improve prediction accuracy, while increasing training cost up to 5x [S1, unofficial].
*   **Parameter Importance:** Removing the volume feature vector decreases accuracy by $\approx 40\times$. Removing image resolution decreases accuracy by $4\times$ [S1, unofficial].

## Operating characteristics
*   **Time Complexity:** VolumeNet inference is $< 10$ ms; PredNet inference is $< 0.5$ ms [S1, unofficial]. Training VolumeNet takes $\approx 2.5$ hours; PredNet takes $\approx 5$ minutes [S1, unofficial].
*   **Memory Complexity:** Compact feature vector (e.g., dimension 4) significantly reduces memory compared to implicit neural representations or storing full volume weights [S1, unofficial].
*   **Performance:** Achieves high prediction accuracy with small errors ($< 0.02$s) across multiple rendering frameworks (CPU/GPU) and datasets (Nyx, MAESTROeX, Castro, Chameleon) [S1, unofficial].
*   **Strengths:** Fast inference, decoupled volume processing (feature extracted once, reused for multiple camera/TF changes), no assumptions about underlying rendering method or hardware [S1, unofficial].
*   **Limitations:** Requires training data; systematic underprediction for high-cost scenarios; sensitive to feature vector dimension choice [S1, unofficial].
*   **When to Use:** Dynamic parameter adaptation for stable frame rates, load balancing in distributed rendering, interactive exploration where rapid time estimation is needed [S1, unofficial].
*   **When Not to Use:** When training data is unavailable or insufficient to capture the distribution of rendering scenarios; when extreme accuracy for rare, high-cost peaks is critical without addressing data imbalance [S1, unofficial].

## Background
ENTIRE was introduced by Z. Yin, H. Gadirov, J. Kosinka, and S. Frey from the University of Groningen in 2026 [S1, unofficial]. It is the first deep learning-based approach for dynamic volume rendering time prediction [S1, unofficial]. The method was evaluated on datasets from scientific simulations (Nyx, MAESTROeX, Castro, Chameleon) and rendering frameworks (CUDA raycaster, yt renderer) [S1, unofficial]. It addresses the limitations of prior statistical methods that required manual modeling or assumed specific rendering inner workings [S1, unofficial].

## Sources
[S1] ENTIRE: Learning-based Volume Rendering Time Prediction, https://doi.org/10.48550/arxiv.2501.12119, fetched 2026-10-26 (unofficial)
