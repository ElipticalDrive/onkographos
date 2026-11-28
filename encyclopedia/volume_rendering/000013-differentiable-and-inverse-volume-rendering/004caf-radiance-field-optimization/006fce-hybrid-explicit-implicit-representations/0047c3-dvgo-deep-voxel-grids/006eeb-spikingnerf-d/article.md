# SpikingNeRF-D
Also called: DVGO-based SpikingNeRF, SpikingNeRF based on DVGO

## Overview
SpikingNeRF-D is a differentiable volume rendering method that integrates spiking neural network (SNN) principles into the Deep Voxel Grids (DVGO) framework [S1, unofficial]. It replaces the standard density Multi-Layer Perceptron (MLP) with explicit voxel grids for density and feature storage, while using a spiking MLP (sMLP) to predict RGB colors from the features. This approach aims to improve hardware efficiency by producing regular, condensed data structures suitable for neuromorphic hardware and DNN accelerators, distinguishing it from standard NeRF variants that rely on dense MLP queries for all volumetric parameters [S1, unofficial].

## How it works
**Inputs:**
*   Density voxel grid $V_\sigma$ and feature voxel grid $V_f$.
*   Spiking MLP $sMLP(\cdot)$.
*   Camera view direction $v$.
*   Rays $R^N = \{r_1, r_2, ..., r_N\}$ from camera origins to pixel directions for $N$ pixels.
*   Number of sampled points per ray $M$.
*   Ground-truth RGB images $C = \{C_1, C_2, ..., C_N\}$ (for training).

**Outputs:**
*   Rendered RGB image $\hat{C} = \{\hat{C}_1, \hat{C}_2, ..., \hat{C}_N\}$.
*   Training loss $L$.

**Data Structures:**
*   $V_\sigma$: 3D voxel grid storing learnable density values.
*   $V_f$: 3D voxel grid storing learnable feature vectors.
*   $P^{N \times M}$: Tensor of sampled 3D coordinates.
*   $\alpha^{N \times M}$: Tensor of opacity weights.
*   $w^{N \times M}$: Tensor of accumulation weights.
*   $P'$: Filtered set of coordinates (variable length per ray).
*   $T$: Temporal length, defined as the maximum number of points among the batched rays after filtering.

**Parameters:**
*   $N$: Number of pixels in the scene.
*   $M$: Number of sampled points per ray.
*   $T$: Temporal length (dynamic, based on batch).
*   Typical values for $N$ and $M$ are not specified in the sources [S1, unofficial].

**Algorithm Steps:**
1.  **Sampling:** Sample $M$ points along each of the $N$ rays to form the coordinate tensor $P^{N \times M} = \{p_{1,1}, p_{1,2}, ..., p_{N,M}\}$ [S1, unofficial].
2.  **Density Query & Weighting:** Query $V_\sigma$ at coordinates $P$ to compute opacity weights $\alpha^{N \times M}$ and accumulation weights $w^{N \times M}$. The specific formulas for $\alpha$ and $w$ are referenced as Eq. 6 and Eq. 3 in the source, which correspond to standard volumetric rendering weight calculations based on density [S1, unofficial].
3.  **Masking:** Filter $P$ using $\alpha$ and $w$ to remove irrelevant points (e.g., those with negligible contribution), resulting in filtered coordinates $P'$ [S1, unofficial]. The masking logic is referenced as Eq. 8.
4.  **Feature Extraction:** Query $V_f$ at coordinates $P'$ and incorporate view direction $v$ to extract volumetric parameters $InputMLP$. This step is referenced as Eq. 7 and Eq. 2 [S1, unofficial].
5.  **Temporal Length Calculation:** Determine $T$, the maximum number of points among the batched rays in $P'$ [S1, unofficial].
6.  **Transformation:** Apply Temporal Padding (TP) or Temporal Condensing with Padding (TCP) to $InputMLP$ to create $InputsMLP$ with shape compatible for the sMLP. This transformation handles the variable number of points per ray by padding or condensing them to the fixed length $T$. This is referenced as Eq. 14 and Section TCP [S1, unofficial].
7.  **Color Prediction:** Pass $InputsMLP$ through the spiking MLP $sMLP$ to compute RGB values $c^{N \times T}$ [S1, unofficial].
8.  **Accumulation:** Accumulate the RGB values $c$ using the weights $\alpha$ and $w$ and filtered coordinates $P'$ to produce the final rendered image $\hat{C}$. This follows standard volume rendering integration, referenced as Eq. 9 [S1, unofficial].
9.  **Loss Calculation:** If training, calculate the Mean Squared Error (MSE) loss $L$ between the rendered image $\hat{C}$ and ground truth $C$, referenced as Eq. 5 [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def SpikingNeRF_D(V_sigma, V_f, sMLP, v, R, M, C_gt):
    """
    V_sigma: Density voxel grid
    V_f: Feature voxel grid
    sMLP: Spiking MLP function
    v: View direction vector
    R: List of N rays
    M: Number of samples per ray
    C_gt: Ground truth RGB image (N, H, W, 3) or flattened (N, 3)
    """
    N = len(R)
    
    # Step 1: Sample points
    # P shape: (N, M, 3)
    P = Sample(R, M)
    
    # Step 2: Query density and compute weights
    # alpha shape: (N, M), w shape: (N, M)
    # Note: Exact formulas for alpha and w depend on V_sigma interpolation
    # and standard volume rendering equations (Eq. 6, Eq. 3 in source)
    alpha, w = Weigh(P, V_sigma)
    
    # Step 3: Mask irrelevant points
    # P_prime is a list of arrays, where P_prime[i] contains points for ray i
    # that have significant weight. Length varies per ray.
    P_prime = Mask(P, alpha, w)
    
    # Step 4: Extract features for filtered points
    # InputMLP contains features for all points in P_prime
    # Shape is variable, handled by subsequent transformation
    InputMLP = ExtractFeatures(P_prime, V_f, v)
    
    # Step 5: Determine temporal length T
    # T is the max number of points across all rays in the batch
    T = max(len(pts) for pts in P_prime)
    
    # Step 6: Transform InputMLP to fixed size T
    # Uses TP (Temporal Padding) or TCP (Temporal Condensing with Padding)
    # InputsMLP shape: (N, T, feature_dim)
    InputsMLP = Transform(InputMLP, T, mode='TCP')
    
    # Step 7: Predict RGB values using sMLP
    # c shape: (N, T, 3)
    c = sMLP(InputsMLP)
    
    # Step 8: Accumulate RGB values to form final image
    # Re-associate weights with filtered points for accumulation
    # C_hat shape: (N, 3)
    C_hat = Accumulate(P_prime, alpha, w, c)
    
    # Step 9: Calculate loss
    L = MSE(C_hat, C_gt)
    
    return C_hat, L
```

**Edge Cases and Pitfalls:**
*   **Variable Point Counts:** The number of valid points per ray varies after masking. The TP or TCP transformation is required to handle this irregularity for the sMLP [S1, unofficial].
*   **Hardware Overhead:** While data condensing incurs overhead, the resulting regular structure benefits hardware accelerators [S1, unofficial].
*   **Dependency on DVGO:** The method relies on the specific voxel grid querying mechanisms of DVGO [S1, unofficial].

## Operating characteristics
*   **Time Complexity:** Dominated by voxel grid queries and sMLP inference. The masking step reduces the number of points processed by the sMLP, potentially saving computation compared to processing all $M$ points.
*   **Memory Complexity:** Requires storage for $V_\sigma$ and $V_f$. The sMLP parameters are stored separately.
*   **Strengths:** Hardware-friendly due to regular data structures after TCP/TP transformation; compatible with neuromorphic hardware [S1, unofficial].
*   **Limitations:** Dependent on the quality of the voxel grid initialization and the sMLP's ability to model complex radiance fields. The masking and transformation steps add computational overhead [S1, unofficial].
*   **When to use:** When hardware efficiency (especially for neuromorphic or specific DNN accelerators) is a priority and the scene can be well-represented by voxel grids [S1, unofficial].

## Background
SpikingNeRF-D was proposed as part of the SpikingNeRF framework, which integrates bio-inspired spiking neural networks into neural radiance fields [S1, unofficial]. It is built upon the DVGO (Deep Voxel Grids) framework by Sun, Sun, and Chen (2022), which accelerates NeRF by using voxel grids for density [S1, unofficial]. The method was designed to leverage the efficiency of spiking networks and regular data structures for better hardware compatibility, including neuromorphic chips [S1, unofficial].

## Sources
[S1] SpikingNeRF: Making Bio-inspired Neural Networks See through the Real World, https://doi.org/10.48550/arxiv.2309.10987, fetched 2026-10-26 (unofficial)
