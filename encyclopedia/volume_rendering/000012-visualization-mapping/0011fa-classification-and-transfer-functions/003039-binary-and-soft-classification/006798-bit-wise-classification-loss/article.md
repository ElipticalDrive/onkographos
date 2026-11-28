# Bit-wise Classification Loss

Also called: Weighted Bit-wise BCE Loss

## Overview
Bit-wise Classification Loss is a training objective for neural volume rendering that treats color prediction as a multi-label binary classification problem rather than a regression task [S1, unofficial]. It decomposes continuous color values into 8-bit binary vectors and computes a weighted Binary Cross Entropy (BCE) for each bit position, prioritizing higher-order bits [S1, unofficial]. This approach addresses the limitations of standard regression losses (like MSE) in neural fields, which can suffer from optimization pitfalls and lack the structural awareness of discrete bit relationships [S1, unofficial]. It is distinct from [Binary Classification](931-binary-classification.md) by using soft probabilistic outputs for all 8 bits simultaneously, and from [Piecewise-Linear Classification](881-piecewise-linear-classification.md) by operating on the final rendered pixel values rather than voxel densities.

## How it works
**Inputs and Outputs:**
*   **Input:** Predicted probability vector $\hat{y} \in [0, 1]^8$ for a single color channel, and ground truth binary vector $y \in \{0, 1\}^8$ derived from the normalized continuous color value $C \in [0, 1]$.
*   **Output:** A scalar loss value $l_b(\hat{y}, y)$ for the channel.

**Data Structures:**
*   **Ground Truth Encoding:** The continuous color value $C$ (normalized from raw integer $y_{raw} \in [0, 255]$) is converted to an 8-bit binary vector $y = [y^{(1)}, y^{(2)}, \dots, y^{(8)}]$ using standard binary representation [S1, unofficial].
*   **Prediction:** The neural network outputs 8 logits per color channel, passed through a sigmoid function to produce $\hat{y}^{(j)}$ for each bit $j$.

**Formula:**
The loss is defined as [S1, unofficial]:

$$
l_b(\hat{y}, y) = \frac{1}{255} \sum_{j=1}^{8} 2^{j-1} \cdot \text{BCE}(\hat{y}^{(j)}, y^{(j)})
$$

Where:
*   $j$ is the bit place index, ranging from 1 (least significant) to 8 (most significant).
*   $\hat{y}^{(j)}$ is the predicted probability for bit $j$.
*   $y^{(j)}$ is the ground truth bit value (0 or 1).
*   $\text{BCE}(\hat{y}^{(j)}, y^{(j)}) = - [y^{(j)} \log(\hat{y}^{(j)}) + (1 - y^{(j)}) \log(1 - \hat{y}^{(j)})]$.
*   The weight $2^{j-1}$ assigns higher importance to higher-order bits, reflecting their place value in the binary number system.
*   The factor $\frac{1}{255}$ normalizes the loss magnitude relative to the raw color range [S1, unofficial].

**Parameters:**
*   **Bit Index $j$:** Fixed range $1 \dots 8$.
*   **Weighting Factor:** $2^{j-1}$.

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def bit_wise_classification_loss(predicted_probs, ground_truth_bits):
    """
    Compute Bit-wise Classification Loss for a single color channel.
    
    Args:
        predicted_probs: List of 8 floats in [0, 1], representing sigmoid outputs for each bit.
        ground_truth_bits: List of 8 integers (0 or 1), derived from the 8-bit binary representation of the target color value.
        
    Returns:
        Scalar loss value.
    """
    total_loss = 0.0
    for j in range(8):
        # j is 0-indexed in loop, corresponds to bit index j+1 in formula
        bit_index = j + 1
        weight = 2 ** (bit_index - 1)
        
        y_pred = predicted_probs[j]
        y_true = ground_truth_bits[j]
        
        # Clip predictions to avoid log(0)
        y_pred = max(min(y_pred, 1 - 1e-7), 1e-7)
        
        # Binary Cross Entropy for this bit
        bce = -(y_true * math.log(y_pred) + (1 - y_true) * math.log(1 - y_pred))
        
        total_loss += weight * bce
        
    # Normalize by 255 as per source
    loss = total_loss / 255.0
    return loss

def encode_color_to_bits(color_value_normalized):
    """
    Convert a normalized continuous color value [0, 1] to 8-bit binary vector.
    Assumes color_value_normalized is derived from raw integer [0, 255].
    """
    # Convert normalized value back to integer [0, 255]
    raw_value = int(round(color_value_normalized * 255))
    # Clamp to valid range
    raw_value = max(0, min(255, raw_value))
    
    bits = []
    for i in range(8):
        # Extract bit i (0 is LSB)
        bit = (raw_value >> i) & 1
        bits.append(bit)
    return bits
```

**Edge Cases:**
*   **Precision:** Sigmoid outputs must be clipped before taking logarithms to prevent numerical instability [G].
*   **Normalization:** The ground truth must be correctly normalized from raw integer values to $[0, 1]$ before encoding to ensure the binary representation matches the expected range [S1, unofficial].

## Subtopics

### Binary-Number Target Encoding
A target encoding rule that projects a continuous color value (0-255) into an 8-bit binary vector to serve as labels for a multi-label binary classification task [S1, unofficial]. It replaces naive one-hot encoding (which treats 1 and 255 as equally distant from 0) with a structure that preserves numerical proximity [S1, unofficial]. Choose this when using neural fields for volume rendering to leverage the hierarchical importance of color bits.

## Operating characteristics
*   **Time Complexity:** $O(1)$ per pixel per channel, as it involves a fixed 8-bit summation.
*   **Memory Complexity:** Requires storing 8 logits per channel instead of 1 continuous value, increasing output dimensionality from 3 to 24 logits per sample [S1, unofficial].
*   **Performance:** Empirically shown to significantly improve existing neural field methods compared to standard MSE regression [S1, unofficial].
*   **Strengths:** Avoids pitfalls of regression formulations in neural fields; respects the hierarchical nature of color values; potentially more stable optimization [S1, unofficial].
*   **Limitations:** Requires decoding predicted probabilities back to continuous values for comparison or rendering if soft labels are used; increased model output size [S1, unofficial].
*   **When to use:** When training neural volume rendering models (like NeRF variants) and standard regression losses fail to converge or produce poor results.
*   **When not to use:** If computational resources are extremely constrained and the overhead of 24 logits per sample is prohibitive, or if the rendering pipeline does not support probabilistic outputs.

## Background
Originated in the paper "Neural Field Classifiers via Target Encoding and Classification Loss" (ICLR 2024) [S1, unofficial]. The work challenges the assumption that regression formulations are superior for neural field methods, proposing classification-based alternatives with specific target encodings and loss functions [S1, unofficial]. It is used in the context of neural volume rendering and neural fields.

## Sources
[S1] Neural Field Classifiers via Target Encoding and Classification Loss, https://doi.org/10.48550/arxiv.2403.01058, fetched 2026-10-26 (unofficial)
