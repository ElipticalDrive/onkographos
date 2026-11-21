# MIMO-DONeRF

Also called: MIMO-NeRF applied to DONeRF

## Overview
MIMO-DONeRF is a method that applies the Multi-Input Multi-Output (MIMO) neural rendering concept to the DONeRF architecture [S1]. It modifies the shading network to accept multiple spatial inputs and view directions simultaneously, producing multiple color and density outputs per forward pass [S1]. This approach improves inference speed compared to increasing the number of samples in standard DONeRF, while maintaining or improving image quality by allowing the network to accumulate neighbor information through grouped samples [S1]. It differs from standard DONeRF by processing $N_p$ samples in parallel rather than sequentially, and from MIMO-NeRF by being specifically adapted to the two-network structure (depth oracle and shading) of DONeRF [S1].

## How it works
**Inputs and Outputs:**
The method operates on rays. For each ray, a depth oracle network selects $N_s$ samples [S1]. The MIMO shading network takes these $N_s$ samples, grouped into batches of size $N_p$, as input. It outputs colors and densities for all grouped samples [S1].

**Data Structures:**
The shading network accepts a batch of inputs. For a group of $N_p$ samples, the input consists of positions $(x_i, \dots, x_j)$ and a single view direction $d$, where $j = i + N_p - 1$ [S1]. The output consists of colors $(c_i, \dots, c_j)$ and densities $(\sigma_i, \dots, \sigma_j)$ for the same indices [S1]. The network architecture (hidden unit dimensions, number of layers, activation functions) remains identical to the original DONeRF shading network [S1].

**Number of Runs:**
The total number of forward passes (# Run) required to process all $N_s$ samples is calculated as:
$$ \text{\# Run} = 1 + \frac{N_s}{N_p} $$
where $N_s$ is the number of samples selected by the depth oracle and $N_p$ is the number of grouped samples processed in parallel [S1]. In standard DONeRF, $N_p=1$, so # Run equals $N_s$ [S1].

**Parameters:**
- $N_s$: Number of samples selected by the depth oracle network. Typical values: 4, 5, 11, 16 [S1].
- $N_p$: Number of grouped samples (parallel inputs) processed by the shading network. Typical values: 1, 4 [S1].
- $\lambda$: Hyperparameter for the MIMO loss function. Set to 0.001 [S1].

**Training and Loss:**
The depth oracle and shading networks are trained separately for 300,000 iterations using the Adam optimizer [S1]. The learning rate is $5 \times 10^{-4}$ with momentum terms $\beta_1=0.9$ and $\beta_2=0.999$ [S1]. The batch size is 4096 rays [S1]. MIMO-DONeRF uses a specific loss function $L_{\text{MIMO}}$ (Equation 10 in the source) instead of the standard pixel loss $L_{\text{pixel}}$ (Equation 3 in the source) [S1]. The implementation is based on the official DONeRF source code [S1].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def mimo_donerrf_shading_network_forward(positions, view_direction):
    """
    Forward pass of the MIMO-DONeRF shading network.
    
    Args:
        positions: Tensor of shape (N_p, 3) containing spatial coordinates 
                   (x_i, ..., x_j) for N_p grouped samples.
        view_direction: Tensor of shape (3) containing the view direction d.
        
    Returns:
        colors: Tensor of shape (N_p, 3) containing colors (c_i, ..., c_j).
        densities: Tensor of shape (N_p) containing densities (sigma_i, ..., sigma_j).
    """
    # Apply positional encoding to positions to get 63-dimensional vector gamma(x)
    # Apply positional encoding to view direction to get 27-dimensional vector gamma(d)
    # Note: The source specifies the depth oracle architecture details, but states
    # the shading network has the "same network architecture" as DONeRF.
    # Standard DONeRF shading network uses positional encoding.
    
    # Input to first layer: gamma(x)
    # Intermediate feature vector is computed through hidden layers.
    # Before the last layer, gamma(d) is concatenated to the feature vector.
    
    # Output layer produces N_p colors and N_p densities.
    # The network processes all N_p inputs in a single forward pass.
    
    # Implementation details of the specific MLP layers are identical to DONeRF.
    # [G] Standard DONeRF shading network uses ReLU activations and a specific
    # number of hidden units (e.g., 256) and layers (e.g., 8). 
    # The output layer has 4 channels (3 for color, 1 for density) per sample.
    
    # Placeholder for the actual DONeRF shading network forward pass logic
    # which is identical to the single-sample case but handles batched inputs.
    colors, densities = donerrf_shading_network(positions, view_direction)
    
    return colors, densities

def compute_mimo_loss(predicted_colors, ground_truth_colors, lambda_param):
    """
    Computes the MIMO loss function.
    
    Args:
        predicted_colors: Tensor of predicted colors.
        ground_truth_colors: Tensor of ground truth colors.
        lambda_param: Hyperparameter lambda (0.001).
        
    Returns:
        loss: Scalar loss value.
    """
    # The source cites Equation 10 for L_MIMO.
    # The exact formula for Equation 10 is not provided in the text snippets.
    # [G] Typically, MIMO losses involve a combination of pixel-wise loss and
    # a regularization term for the multi-output consistency or neighbor 
    # information accumulation. Without the explicit formula, we cannot 
    # implement the exact loss.
    # However, the source states L_MIMO is used as an alternative to L_pixel.
    # L_pixel is typically Mean Squared Error (MSE).
    
    # Since the exact formula for L_MIMO is not in the provided text,
    # we cannot write the code for it.
    # Comment: Implement Equation 10 from the source paper.
    pass
```

**Edge Cases and Pitfalls:**
- If $N_p$ does not divide $N_s$ evenly, the last group may have fewer than $N_p$ samples. The source does not specify how this is handled, but standard batching techniques (padding or variable batch sizes) would be required [G].
- The source notes that DONeRF-4 sometimes yields artifacts (e.g., belt in 'Bulldozer', hair in 'Forest'), which are mitigated by increasing samples to 16 or using MIMO-DONeRF [S1].

## Operating characteristics
**Time and Memory Complexity:**
MIMO-DONeRF reduces the number of forward passes (# Run) compared to standard DONeRF. For $N_s=16$ and $N_p=4$, # Run is $1 + 16/4 = 5$, whereas standard DONeRF requires 16 passes [S1]. This results in a significant speedup. DONeRF-16 increases inference time approximately three times compared to DONeRF-4, while MIMO-DONeRF-16/4-naive and MIMO-DONeRF-16/4-self only increase inference time 1.1 times compared to DONeRF-4 [S1].

**Measured Performance:**
- **PSNR:** MIMO-DONeRF-16/4-self achieves an average PSNR of 36.27, comparable to DONeRF-16 (36.98) and better than DONeRF-4 (31.14) [S1].
- **FLIP:** MIMO-DONeRF-16/4-self achieves an average FLIP of 0.061, comparable to DONeRF-16 (0.062) and better than DONeRF-4 (0.070) [S1].
- **Scene Specifics:** MIMO-DONeRF-16/4-naive and MIMO-DONeRF-16/4-self succeed in representing lotus leaves in the 'Pavillon' scene, whereas DONeRF-16 and DONeRF-4 fail [S1].

**Strengths and Limitations:**
- **Strengths:** Faster inference than increasing $N_s$ in standard DONeRF; better handling of fine details (e.g., lotus leaves) due to neighbor information accumulation; maintains high image quality (PSNR/FLIP) comparable to high-sample DONeRF [S1].
- **Limitations:** Requires modification of the shading network to handle multiple inputs/outputs; relies on the MIMO loss function which may require careful tuning (though $\lambda=0.001$ is suggested) [S1].

**When to Use:**
Use MIMO-DONeRF when inference speed is critical but high image quality (comparable to high-sample DONeRF) is required, particularly for scenes with fine details that suffer from artifacts in low-sample DONeRF [S1].

## Background
MIMO-DONeRF was introduced in the paper "MIMO-NeRF: Fast Neural Rendering with Multi-input Multi-output Neural Radiance Fields" [S1]. It is an application of the MIMO-NeRF concept to the DONeRF architecture [S1]. The method was implemented based on the official DONeRF source code [S1]. The authors note that the depth oracle network in DONeRF is already fast and runs only once per ray, so the MIMO concept is applied to the shading network to improve efficiency [S1].

## Sources
[S1] MIMO-NeRF: Fast Neural Rendering with Multi-input Multi-output Neural Radiance Fields, https://arxiv.org/abs/2310.01821v1, fetched 2026-10-26
