# Non-Rigid Refinement

Also called: iterative flow network

## Overview
Non-Rigid Refinement is a post-processing alignment step that uses an iterative optical flow network to correct non-rigid deformations or misalignments in NeRF features by warping neighbor features to test features [S1]. It addresses the domain gap between blurry rendered novel views and sharp neighboring images, which causes standard 3D alignment to fail due to camera pose errors and geometry estimation inaccuracies [S1]. Unlike general-purpose alignment, this method is trained end-to-end specifically for the task of refining feature correspondences in the presence of these domain gaps [S1].

## How it works
**Inputs and Outputs:**
*   **Input:** A pair of feature maps: Neighbour Features ($H^f_{i\rightarrow k}$) and Test Features ($I^f_k$). The neighbour features are the reprojected features from adjacent views, and the test features are from the novel view [S1].
*   **Output:** Refined Neighbour Features ($H^{f'}_{i\rightarrow k}$) and an estimated 2D optical flow field [S1].

**Data Structures:**
*   **Feature Maps:** 2D tensors representing features extracted from images [S1]. The specific channel depth is not explicitly defined in the source, but the architecture processes them as multi-channel inputs [S1].
*   **Flow Field:** A 2-channel tensor representing the horizontal and vertical displacement vectors for each pixel [S1].

**Algorithm Steps:**
1.  **Resolution Reduction:** The input feature maps are downsampled by a factor of 8 [S1]. The optical flow estimation is performed at this 8x downsampled resolution [S1].
2.  **Flow Estimation:** The downsampled neighbour features and test features are passed through the optical flow network $\mathcal{F}$ to estimate the flow field $f_{H^f_{i\rightarrow k}}$ [S1]:
    $$ f_{H^f_{i\rightarrow k}} = \mathcal{F}(H^f_{i\rightarrow k}, I^f_k) $$
3.  **Warping:** The neighbour features are warped using the estimated 2D flow field via a 2D warping function $warp_{2D}$ [S1]:
    $$ H^{f'}_{i\rightarrow k} = warp_{2D}(H^f_{i\rightarrow k}, f_{H^f_{i\rightarrow k}}) $$
4.  **Iteration:** The estimation and warping process is iterated 32 times to refine the alignment progressively [S1].

**Network Architecture ($\mathcal{F}$):**
The network $\mathcal{F}$ is a lightweight convolutional neural network [S1]. The architecture is identical to the original formulation in [6] but adapted to accept multi-channel features instead of 3-channel RGB images [S1]. The layers are as follows:

1.  Conv2d: kernel size 7, 64 channels, stride 4. Followed by ReLU.
2.  Conv2d: kernel size 3, 64 channels. Followed by ReLU.
3.  Conv2d: kernel size 3, 64 channels. Skip connection added to previous output [S1]. Followed by ReLU.
4.  Conv2d: kernel size 3, 64 channels, stride 2. Followed by ReLU.
5.  Conv2d: kernel size 3, 64 channels. Skip connection added to previous output [S1]. Followed by ReLU.
6.  Conv2d: kernel size 3, 64 channels. Followed by ReLU.
7.  Conv2d: kernel size 3, 64 channels. Followed by ReLU.
8.  Conv2d: kernel size 3, 128 channels. Followed by ReLU.
9.  Conv2d: kernel size 3, 64 channels. Followed by ReLU.
10. Conv2d: kernel size 3, 2 channels (representing the optical flow vector). Followed by ReLU.

**Upsampling:**
The estimated flow field from the network is at the reduced resolution [S1]. It is upsampled to full resolution using bilinear upsampling before being used in the warping step [S1].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import torch
import torch.nn as nn
import torch.nn.functional as F

class NonRigidRefinementNetwork(nn.Module):
    def __init__(self, in_channels):
        super(NonRigidRefinementNetwork, self).__init__()
        # Layer 1: Conv2d k7 c64 s4 -> ReLU
        self.conv1 = nn.Conv2d(in_channels, 64, kernel_size=7, stride=4, padding=3)
        self.relu1 = nn.ReLU()
        
        # Layer 2: Conv2d k3 c64 -> ReLU
        self.conv2 = nn.Conv2d(64, 64, kernel_size=3, padding=1)
        self.relu2 = nn.ReLU()
        
        # Layer 3: Conv2d k3 c64 + skip -> ReLU
        self.conv3 = nn.Conv2d(64, 64, kernel_size=3, padding=1)
        self.relu3 = nn.ReLU()
        
        # Layer 4: Conv2d k3 c64 s2 -> ReLU
        self.conv4 = nn.Conv2d(64, 64, kernel_size=3, stride=2, padding=1)
        self.relu4 = nn.ReLU()
        
        # Layer 5: Conv2d k3 c64 + skip -> ReLU
        self.conv5 = nn.Conv2d(64, 64, kernel_size=3, padding=1)
        self.relu5 = nn.ReLU()
        
        # Layer 6: Conv2d k3 c64 -> ReLU
        self.conv6 = nn.Conv2d(64, 64, kernel_size=3, padding=1)
        self.relu6 = nn.ReLU()
        
        # Layer 7: Conv2d k3 c64 -> ReLU
        self.conv7 = nn.Conv2d(64, 64, kernel_size=3, padding=1)
        self.relu7 = nn.ReLU()
        
        # Layer 8: Conv2d k3 c128 -> ReLU
        self.conv8 = nn.Conv2d(64, 128, kernel_size=3, padding=1)
        self.relu8 = nn.ReLU()
        
        # Layer 9: Conv2d k3 c64 -> ReLU
        self.conv9 = nn.Conv2d(128, 64, kernel_size=3, padding=1)
        self.relu9 = nn.ReLU()
        
        # Layer 10: Conv2d k3 c2 -> ReLU
        self.conv10 = nn.Conv2d(64, 2, kernel_size=3, padding=1)
        self.relu10 = nn.ReLU()

    def forward(self, neighbour_features, test_features):
        # Concatenate inputs along channel dimension
        x = torch.cat([neighbour_features, test_features], dim=1)
        
        # Layer 1
        x = self.relu1(self.conv1(x))
        
        # Layer 2
        x = self.relu2(self.conv2(x))
        
        # Layer 3 (Skip connection)
        x_skip = x
        x = self.relu3(self.conv3(x) + x_skip)
        
        # Layer 4
        x = self.relu4(self.conv4(x))
        
        # Layer 5 (Skip connection)
        x_skip = x
        x = self.relu5(self.conv5(x) + x_skip)
        
        # Layer 6
        x = self.relu6(self.conv6(x))
        
        # Layer 7
        x = self.relu7(self.conv7(x))
        
        # Layer 8
        x = self.relu8(self.conv8(x))
        
        # Layer 9
        x = self.relu9(self.conv9(x))
        
        # Layer 10
        flow = self.relu10(self.conv10(x))
        return flow

def warp_2d(features, flow):
    """
    Warps the input features using the provided 2D flow field via bilinear sampling [G].
    """
    B, C, H, W = features.shape
    # Create normalized grid coordinates [-1, 1]
    y, x = torch.meshgrid(torch.linspace(-1, 1, H), torch.linspace(-1, 1, W), indexing='ij')
    grid = torch.stack([x, y], dim=-1).to(features.device)
    # Flow is in pixel coordinates; convert to normalized [-1, 1]
    flow_norm = flow.permute(0, 2, 3, 1) * 2.0 / max(H, W)
    grid = grid + flow_norm
    grid = torch.clamp(grid, -1, 1)
    return F.grid_sample(features, grid, mode='bilinear', padding_mode='zeros')

def non_rigid_refinement(neighbour_features, test_features, num_iterations=32):
    """
    Performs non-rigid refinement iteratively at 8x downsampled resolution,
    then warps the full-resolution features with the final upsampled flow.
    """
    h, w = neighbour_features.shape[2], neighbour_features.shape[3]
    small_h, small_w = h // 8, w // 8
    
    # Downsample inputs by 8x
    small_neighbour = F.interpolate(neighbour_features, size=(small_h, small_w), mode='bilinear', align_corners=False)
    small_test = F.interpolate(test_features, size=(small_h, small_w), mode='bilinear', align_corners=False)
    
    flow_net = NonRigidRefinementNetwork(in_channels=small_neighbour.shape[1])
    current_features = small_neighbour
    
    # Iterative refinement loop at reduced resolution
    for _ in range(num_iterations):
        flow_small = flow_net(current_features, small_test)
        current_features = warp_2d(current_features, flow_small)
        
    # Final flow estimation and upsampling to full resolution
    final_flow_small = flow_net(current_features, small_test)
    final_flow_full = F.interpolate(final_flow_small, size=(h, w), mode='bilinear', align_corners=False)
    
    # Warp original full-resolution neighbour features
    refined_features = warp_2d(neighbour_features, final_flow_full)
    
    return refined_features, final_flow_full
```

**Parameters:**
*   **Optical flow estimation resolution:** 8x downsampled relative to the input feature resolution [S1].
*   **Number of iterations:** 32 [S1].

**Edge Cases and Pitfalls:**
*   **Domain Gap:** Standard optical flow methods expect clean images, but NeRF renders can be blurry, creating a domain gap [S1]. This method is specifically trained to handle this [S1].
*   **Camera Pose Errors:** Errors in COLMAP pose estimation can lead to misalignments that 3D alignment cannot fully correct [S1].

## Operating characteristics
*   **Time Complexity:** Not given in the sources. The method involves 32 iterations of a lightweight convolutional network and warping operations [S1].
*   **Memory Complexity:** Not given in the sources. Requires storing the feature maps and the flow field [G].
*   **Strengths:** Effectively reduces ghosting artifacts caused by misalignment from camera pose errors and non-rigid motion [S1]. It handles the domain gap between blurry rendered images and sharp neighbours better than general-purpose optical flow [S1].
*   **Limitations:** Relies on the quality of the initial 3D alignment and feature extraction. Performance depends on the specific training of the flow network [S1].
*   **When to use:** When there are significant misalignments due to camera pose errors or non-rigid motion that 3D alignment cannot resolve, and when the domain gap between rendered and real images is significant [S1].

## Background
Non-Rigid Refinement was introduced in the RoGUENeRF paper by S. Catley-Chandar et al. [S1]. It builds upon the Iterative Alignment Strategy [6957-iterative-alignment-strategy.md] and is part of the broader effort to enhance NeRF quality by addressing alignment issues [S1]. The optical flow network architecture is based on the work presented in [6] [S1].

## Sources
[S1] RoGUENeRF: A Robust Geometry-Consistent Universal Enhancer for NeRF, https://doi.org/10.1007/978-3-031-73254-6_4, fetched 2026-10-29
