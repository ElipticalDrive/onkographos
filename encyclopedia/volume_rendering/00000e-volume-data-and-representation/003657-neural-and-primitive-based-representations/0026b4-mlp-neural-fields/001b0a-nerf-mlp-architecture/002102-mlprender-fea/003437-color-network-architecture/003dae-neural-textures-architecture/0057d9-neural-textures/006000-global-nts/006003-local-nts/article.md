# Local NTS

Also called: Local Neural Texture Stack

## Overview
Local NTS is a Neural Texture Stack (NTS) variant that generates view-dependent textures by transforming observation-space UV coordinates into canonical UV coordinates using a pose-conditioned deformation field [S1]. Unlike Global NTS, which conditions texture generation directly on global pose $\boldsymbol{\theta}$, Local NTS relies on local geometric correspondence, making it computationally efficient but topologically rigid [S1]. It is distinguished from its sibling Spatial NTS by its inability to generate new topological space, limiting its capacity to model occlusion-induced texture changes (e.g., belts appearing or disappearing) [S1].

## How it works
**Inputs and Outputs:**
*   **Inputs:** Observation UV coordinates $(\hat{U}_t^k(r), \hat{V}_t^k(r))$, pose $\boldsymbol{\theta}$, part label vector $\boldsymbol{k}$, and viewing direction $\boldsymbol{d}$ [S1].
*   **Outputs:** View-dependent RGB color $\hat{C}_k(r)$ [S1].

**Data Structures:**
1.  **Deformation Field:** A Multi-Layer Perceptron (MLP) consisting of three fully-connected layers, each with 256 channels [S1].
2.  **Color Network:** A subsequent MLP consisting of five fully-connected layers with 256 channels, followed by one fully-connected layer with 128 channels [S1].

**Algorithm:**
The method proceeds in two stages: deformation and color prediction.

1.  **Deformation:** The deformation field takes the observation UV coordinates and the pose $\boldsymbol{\theta}$ as input. It outputs the canonical UV coordinates $(\hat{U}_t^{k0}(r), \hat{V}_t^{k0}(r))$ [S1].
    $$ (\hat{U}_t^{k0}(r), \hat{V}_t^{k0}(r)) = \text{DeformationField}(\hat{U}_t^k(r), \hat{V}_t^k(r), \boldsymbol{\theta}) $$
    [S1]

2.  **Color Prediction:** The Color Network takes the transformed canonical UV coordinates, the part label vector $\boldsymbol{k}$, and the viewing direction $\boldsymbol{d}$ as inputs. The viewing direction $\boldsymbol{d}$ is processed by a positional encoding function $\gamma(\boldsymbol{d})$ and passed through an additional layer with 128 channels [S1]. The final layer applies a sigmoid activation to output the RGB color [S1].
    $$ \hat{C}_k(r) = \text{Sigmoid}(\text{ColorNetwork}(\hat{U}_t^{k0}(r), \hat{V}_t^{k0}(r), \boldsymbol{k}, \gamma(\boldsymbol{d}))) $$
    [S1]

**Parameters:**
*   Deformation Field: 3 layers, 256 channels per layer [S1].
*   Color Network: 5 layers of 256 channels, 1 layer of 128 channels [S1].
*   NTS Resolution: Default 64 × 64; Maximum 128 × 128 [S1].

**Edge Cases and Pitfalls:**
*   **Topological Invariance:** Local NTS cannot generate new topological space. It is incapable of modeling textures that change topology due to occlusion or pose variation, such as a belt appearing when a hand is raised and disappearing when covered by a shirt [S1].
*   **Precision:** Not given in the sources.

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def local_nts_render(obs_uv, pose, part_label, view_dir):
    """
    Renders RGB color using Local Neural Texture Stack.
    
    Args:
        obs_uv: Tuple (U, V) of observation UV coordinates.
        pose: Pose vector theta.
        part_label: Part label vector k.
        view_dir: Viewing direction vector d.
        
    Returns:
        RGB color vector.
    """
    # Step 1: Deformation Field
    # Transforms observation UV to canonical UV using pose-conditioned MLP
    # Architecture: 3 FC layers, 256 channels each
    canonical_uv = deformation_field(obs_uv, pose)
    
    # Step 2: Color Network
    # Takes canonical UV, part label, and view direction
    # Architecture: 5 FC layers (256 ch), 1 FC layer (128 ch), Sigmoid output
    
    # Process viewing direction with positional encoding
    # Note: Specific positional encoding function is not detailed in sources [G]
    encoded_view = positional_encoding(view_dir)
    
    # Pass through additional layer with 128 channels for view direction
    # Note: The source states "an additional layer with 128 channels" 
    # along with positional encoding. This is integrated into the color network input processing.
    
    # Final color prediction
    # Inputs: canonical_uv, part_label, encoded_view
    # Output: Sigmoid activation
    rgb_color = color_network(canonical_uv, part_label, encoded_view)
    
    return rgb_color

def deformation_field(obs_uv, pose):
    """
    Deformation Field MLP.
    """
    x = obs_uv
    # 3 fully-connected layers of 256 channels
    x = fc_layer_256(x)
    x = fc_layer_256(x)
    x = fc_layer_256(x)
    # Conditioned on pose theta
    # Note: Exact fusion mechanism (concat, add, modulate) not specified in sources [G]
    x = fuse_with_pose(x, pose)
    return x

def color_network(canonical_uv, part_label, encoded_view):
    """
    Color Network MLP.
    """
    # Concatenate inputs
    x = concatenate([canonical_uv, part_label, encoded_view])
    
    # 5 fully-connected layers of 256 channels
    x = fc_layer_256(x)
    x = fc_layer_256(x)
    x = fc_layer_256(x)
    x = fc_layer_256(x)
    x = fc_layer_256(x)
    
    # 1 fully-connected layer of 128 channels
    x = fc_layer_128(x)
    
    # Sigmoid activation
    return sigmoid(x)
```

## Subtopics

### Spatial NTS
A CNN-based Neural Texture Stack that generates a 128-dimensional texture stack $E_i$ using 5 convolution layers [S1]. It captures local 2D texture changes via convolution operations, allowing the subsequent MLP to model only the local mapping between the neural texture stack and RGB color [S1]. It outperforms Global, Local, and Hyper NTS on the CMU dataset (Novel PSNR 30.38, View SSIM 0.966, Synthesis LPIPS 0.036) because it accurately captures high-frequency details like wrinkles and numbers [S1].

### Neural Texture Stack Resolution
The resolution of the NTS grid is a trade-off between quality and memory. The default resolution is 64 × 64, which is recommended as the optimum [S1]. The maximum resolution is 128 × 128 due to memory limitations; while 128 × 128 yields slightly better results, it costs more memory and time without significant quality gain over 64 × 64 [S1].

## Operating characteristics
**Time and Memory Complexity:**
*   Memory usage scales with NTS resolution. 64 × 64 is the recommended default; 128 × 128 is the maximum due to memory constraints [S1].
*   Time complexity increases with resolution, though 128 × 128 is not significantly slower than 64 × 64 in practice [S1].

**Measured Performance:**
*   On the CMU dataset, Local NTS achieves: Novel PSNR 29.82, View SSIM 0.965, Synthesis LPIPS 0.061 [S1].
*   It performs worse than Global NTS (Novel PSNR 29.86), Hyper NTS (Novel PSNR 30.08), and Spatial NTS (Novel PSNR 30.38) [S1].

**Strengths and Limitations:**
*   **Strengths:** Efficient coordinate transformation; suitable for static topologies [S1].
*   **Limitations:** Cannot model topologically varying textures (e.g., occlusion-induced appearance/disappearance of objects like belts) [S1]. It is the most limited among NTS variants [S1].

**When to use:**
*   Use when the texture topology is stable across poses and occlusions are minimal or handled by other means.
*   Avoid when high-frequency local texture details are critical (prefer Spatial NTS) or when topological variations occur (prefer Global or Hyper NTS) [S1].

## Background
Local NTS was proposed in the context of "UV Volumes for Real-time Rendering of Editable Free-view Human Performance" [S1]. It serves as an ablation study component to evaluate the impact of different texture generation strategies (Global, Local, Hyper, Spatial) within the Neural Texture Stack framework [S1]. The method was evaluated on the CMU dataset [S1].

## Sources
[S1] UV Volumes for Real-time Rendering of Editable Free-view Human Performance, https://doi.org/10.1109/cvpr52729.2023.01595, fetched 2026-10-26
