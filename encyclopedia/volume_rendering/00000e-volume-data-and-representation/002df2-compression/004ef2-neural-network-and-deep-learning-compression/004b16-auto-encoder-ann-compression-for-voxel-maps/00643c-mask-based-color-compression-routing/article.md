# Mask-based Color Compression Routing

Also called: Mask-based Color Compression, Multi-path Entropy Module

## Overview
Mask-based Color Compression Routing (MEM) is a module that deduces binary masks to determine whether Gaussian color attributes should be directly quantized for compression or processed through an autoencoder, balancing size and fidelity [S1, unofficial]. It splits color attributes $f_{col}$ into two compression paths ($m=0$ and $m=1$) using a binary mask, where the $m=1$ path routes attributes through an autoencoder for redundancy elimination, while the $m=0$ path compresses them directly under entropy constraints [S1, unofficial]. Unlike methods that degrade parameter values or alter the 3DGS structure, MEM preserves both values and structural integrity, enabling an optimization-free design [S1, unofficial].

## How it works
**Inputs and Outputs:**
*   **Input:** Gaussian color attributes $f_{col}$, entropy constraint parameter $\lambda$.
*   **Output:** Compressed bitstream for $f_{col}$, including mask bits, direct quantization bits ($m=0$), and autoencoder latent bits ($m=1$).

**Data Structures:**
*   **Mask:** Binary vector $M$ of same length as $f_{col}$, where $M_i \in \{0, 1\}$.
*   **Autoencoder Latent:** Quantized latent representation $\hat{y}$ for attributes where $M_i=1$.
*   **Hyperprior:** Coarse representation $\hat{z}$ used to estimate distribution parameters for $\hat{y}$.

**Steps:**
1.  **Mask Generation:** Determine the binary mask $M$ for each color attribute based on the entropy constraint $\lambda$. The mask rate (proportion of $M_i=1$) increases as $\lambda$ increases [S1, unofficial].
2.  **Routing:**
    *   If $M_i = 1$: Route $f_{col}[i]$ to the autoencoder path.
    *   If $M_i = 0$: Route $f_{col}[i]$ to the direct quantization path.
3.  **Autoencoder Path ($m=1$):**
    *   **Analysis Transform:** Map input $x$ (color attribute) into latent space $\hat{y}$ using analysis transform $g_a$ and quantization [S1, unofficial].
    *   **Hyperprior Estimation:** Use coarse representation $\hat{z}$ to estimate distribution parameters of $\hat{y}$ under a Gaussian assumption [S1, unofficial].
    *   **Context Models:** Apply context models to $\hat{y}$ to improve probability estimation [S1, unofficial].
    *   **Entropy Encoding:** Encode $\hat{y}$ using the estimated parameters.
    *   **Synthesis Transform:** Decode $\hat{y}$ and project back to original space as $\hat{x}$ using synthesis transform $g_s$ [S1, unofficial].
    *   **Loss:** Maintain fidelity between $\hat{x}$ and $x$ using rendered images, while minimizing entropy of $\hat{y}$ and $\hat{z}$ [S1, unofficial].
4.  **Direct Quantization Path ($m=0$):**
    *   Quantize $f_{col}[i]$ directly under entropy constraints [S1, unofficial].
5.  **Mask Encoding:** Encode the binary mask $M$. Mask size decreases as mask rate increases because binary distribution entropy drops when one value (1) dominates [S1, unofficial].

**Formulas:**
*   **Compression Ratio:** $32 / \text{bits\_per\_parameter}$ [S1, unofficial].
*   **Loss Function:** Minimizes entropy of $\hat{y}$ and $\hat{z}$ while maintaining fidelity between $\hat{x}$ and $x$ [S1, unofficial].

**Parameters:**
*   $\lambda$: Entropy constraint/regulator controlling mask routing and compression strength. Tested values: $1e-4, 2e-4, 4e-4, 8e-4, 16e-4$ [S1, unofficial].

**Edge Cases and Pitfalls:**
*   **Precision:** Original parameters are stored in float32 (32 bits per parameter) [S1, unofficial].
*   **Geometry Attributes:** $f_{geo}$ is harder to compress due to high variability and low mutual information [S1, unofficial].
*   **Mask Rate Impact:** As $\lambda$ increases, mask rate increases, shifting more $f_{col}$ to $m=1$. Overall $m=1$ storage may increase, but bits per parameter still reduce [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def mask_based_color_compression_routing(f_col, lambda_val):
    """
    Compress color attributes using mask-based routing.
    
    Args:
        f_col: Array of color attributes.
        lambda_val: Entropy constraint parameter.
        
    Returns:
        Compressed bitstream components.
    """
    # Step 1: Generate mask based on lambda
    # Mask rate increases as lambda increases
    mask_rate = get_mask_rate(lambda_val) # Implementation detail not specified in sources
    mask = generate_binary_mask(f_col, mask_rate)
    
    # Step 2: Route attributes
    m1_indices = [i for i, m in enumerate(mask) if m == 1]
    m0_indices = [i for i, m in enumerate(mask) if m == 0]
    
    # Step 3: Process m=1 path (Autoencoder)
    m1_attributes = [f_col[i] for i in m1_indices]
    if m1_attributes:
        # Analysis transform and quantization
        y_hat = analysis_transform_and_quantize(m1_attributes)
        
        # Hyperprior estimation
        z_hat = get_coarse_representation(y_hat)
        distribution_params = estimate_gaussian_params(z_hat)
        
        # Context models and entropy encoding
        m1_bits = entropy_encode(y_hat, distribution_params)
        
        # Synthesis transform (for fidelity check during training)
        x_hat = synthesis_transform(y_hat)
        # Loss calculation would happen here during training
    else:
        m1_bits = []
        
    # Step 4: Process m=0 path (Direct Quantization)
    m0_attributes = [f_col[i] for i in m0_indices]
    if m0_attributes:
        m0_bits = direct_quantize(m0_attributes, lambda_val)
    else:
        m0_bits = []
        
    # Step 5: Encode mask
    mask_bits = encode_binary_mask(mask)
    
    return mask_bits, m0_bits, m1_bits

def get_mask_rate(lambda_val):
    """
    Returns mask rate based on lambda.
    Values from [S1, unofficial]:
    lambda=1e-4 -> 0.62
    lambda=2e-4 -> 0.67
    lambda=4e-4 -> 0.74
    lambda=8e-4 -> 0.82
    lambda=16e-4 -> 0.89
    """
    if lambda_val == 1e-4: return 0.62
    elif lambda_val == 2e-4: return 0.67
    elif lambda_val == 4e-4: return 0.74
    elif lambda_val == 8e-4: return 0.82
    elif lambda_val == 16e-4: return 0.89
    else: return 0.62 # Default or interpolation not specified [G]

def generate_binary_mask(f_col, mask_rate):
    """
    Generates binary mask with specified rate.
    Implementation detail not specified in sources [G]
    """
    n = len(f_col)
    num_ones = int(n * mask_rate)
    mask = [1] * num_ones + [0] * (n - num_ones)
    # Shuffle or assign based on some criterion not specified [G]
    return mask

def analysis_transform_and_quantize(attributes):
    """
    Maps input to latent space and quantizes.
    Uses simple MLPs as per [S1, unofficial]
    """
    # Placeholder for actual network forward pass
    # y = g_a(x)
    # y_hat = quantize(y)
    return quantize(attributes) # Simplified [G]

def get_coarse_representation(y_hat):
    """
    Generates coarse representation z_hat.
    """
    # Placeholder for hyperprior network
    return y_hat # Simplified [G]

def estimate_gaussian_params(z_hat):
    """
    Estimates distribution parameters under Gaussian assumption.
    """
    # Placeholder for parameter estimation
    return {'mean': 0, 'var': 1} # Simplified [G]

def entropy_encode(y_hat, params):
    """
    Encodes y_hat using estimated parameters.
    """
    # Placeholder for arithmetic encoding
    return len(y_hat) * 0.45 # Simplified based on [S1, unofficial] bits/param

def direct_quantize(attributes, lambda_val):
    """
    Directly quantizes attributes under entropy constraints.
    """
    # Placeholder for direct quantization
    return len(attributes) * 10 # Simplified [G]

def encode_binary_mask(mask):
    """
    Encodes the binary mask.
    """
    # Placeholder for mask encoding
    return len(mask) * 0.1 # Simplified [G]
```

## Operating characteristics
*   **Time and Memory Complexity:** Not given in the sources.
*   **Measured Performance:** On DL3DV-GS dataset, total size drops from 27.44 MB ($\lambda=1e-4$) to 15.83 MB ($\lambda=16e-4$) [S1, unofficial]. Mask rate increases from 0.62 to 0.89 [S1, unofficial]. $f_{col}(m=1)$ bits/param drop from 0.45 to 0.36 [S1, unofficial]. $f_{geo}$ bits/param drop from 6.94 to 5.15 [S1, unofficial].
*   **Scaling:** As $\lambda$ increases, mask rate increases, shifting more $f_{col}$ to $m=1$. Overall $m=1$ storage may increase, but bits per parameter still reduce [S1, unofficial].
*   **Strengths:** Preserves parameter values and 3DGS structure integrity, enabling optimization-free design [S1, unofficial]. $f_{col}(m=1)$ is easiest to compress due to latent space operation [S1, unofficial].
*   **Limitations:** $f_{geo}$ is hardest to compress due to high variability and low mutual information [S1, unofficial].
*   **When to Use:** When optimization-free compression is required and preserving structure is critical [S1, unofficial].
*   **When Not to Use:** When per-scene adaptation is beneficial and optimization time is acceptable [S1, unofficial].

## Background
*   **Origin:** Proposed in "Fast Feedforward 3D Gaussian Splatting Compression" [S1, unofficial].
*   **History:** Inspired by image compression techniques, where input attributes are mapped to latent space for compression [S1, unofficial].
*   **Usage:** Used in 3D Gaussian Splatting compression to balance size and fidelity without altering the 3DGS structure [S1, unofficial].

## Sources
[S1] Fast Feedforward 3D Gaussian Splatting Compression, https://doi.org/10.48550/arxiv.2410.08017, fetched 2026-10-26 (unofficial)
