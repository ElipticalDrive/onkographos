# Lossless TSDF Sign Compression

Also called: Topology mask compression

## Overview
Lossless TSDF Sign Compression is a method for compressing implicit volume representations by separately encoding the topological structure (signs) and geometric magnitude (values) of a Truncated Signed Distance Field (TSDF) [S1]. It solves the problem of topological errors in reconstructed surfaces caused by quantization noise in the magnitude values [S1]. Unlike methods that compress the full TSDF field uniformly, this approach ensures that the zero-crossing surface topology remains identical to the ground truth by losslessly encoding the sign configuration, thereby bounding the reconstruction error by the voxel size [S1].

## How it works
The method operates on a block of TSDF data $x$ and produces a compressed bitstream consisting of a quantized latent code $\hat{z}$ and a lossless sign mask $s$ [S1].

**Inputs and Outputs:**
*   Input: A block of TSDF values $x$.
*   Output: Bitstreams $\hat{z}_{bits}$ and $s_{bits}$ [S1].

**Data Structures:**
1.  **Latent Code $\hat{z}$**: The lossily quantized representation of $x$ obtained from an encoder $E$ [S1].
2.  **Signs $s$**: A binary configuration indicating the sign (positive/negative) of each voxel in the ground truth TSDF block [S1].
3.  **Conditional Probability Model $p_{s|\hat{z}}$**: A distribution parameterized by $\theta_s$ that predicts the probability of each sign bit given the latent code $\hat{z}$ [S1].

**Steps:**
1.  **Encode Latent Code**: Compute the lossily quantized latent representation $\hat{z} = b_E(x; \theta_e)$ using the learned encoder $E$ with parameters $\theta_e$ [S1].
2.  **Compute Conditional Distribution**: Using the quantized latent code $\hat{z}$, compute the conditional probability distribution over the TSDF signs $s$ as $p_{s|\hat{z}}(s|\hat{z}; \theta_s)$ [S1].
3.  **Lossless Coding**: Use an entropy coder to losslessly code the signs $s$ using the conditional distribution $p_{s|\hat{z}}(s|\hat{z})$ and the latent code $\hat{z}$ to generate the bitstreams $\hat{z}_{bits}$ and $s_{bits}$ [S1].

**Formulas:**
The reconstruction error $D_{\hat{x}}(x, \hat{x}; \theta_e, \theta_d)$ is upper bounded by the voxel size. For example, with a voxel size of 5mm, the error is bounded by 5mm [S1].

**Parameters:**
*   Voxel size: 5mm (example value) [S1].

**Edge Cases and Pitfalls:**
Without lossless sign compression, the reconstructed surface deviates from the ground truth due to compression error in magnitudes, causing topological errors [S1]. The method relies on the assumption that the sign configuration is critical for topology, while magnitude errors are tolerable within the voxel size [S1].

## Subtopics

### ZipGAN
ZipGAN reframes compression as super-resolution recovery by downsampling a volume and training a super-resolution GAN to reconstruct the full-resolution field [S2, unofficial]. It uses a discriminator-derived quality score as a proxy for reconstruction confidence without access to the original data [S2, unofficial]. Transfer learning from lower to higher compression ratios halves training time at each doubling of $n\delta$ [S2, unofficial]. Compression ratios range from 8 to 4,096 in 3D [S2, unofficial]. On homogeneous isotropic turbulence at $1024^3$ ($Re_\lambda \approx 210$), it maintained high fidelity up to 512× compression and outperformed discrete wavelet transform (DWT) at all ratios above 8× [S2, unofficial]. No formal error bound is guaranteed [S2, unofficial].

## Operating characteristics
The method ensures topological correctness by bounding the reconstruction error by the voxel size [S1]. However, the specific time and memory complexity, measured performance, and scaling characteristics are not given in the sources. The reliance on an entropy coder and a conditional probability model implies computational overhead during encoding and decoding compared to simple quantization [S1].

## Background
Lossless TSDF Sign Compression was introduced in the context of Deep Implicit Volume Compression [S1]. It addresses the limitation of standard TSDF compression where magnitude quantization leads to topological inaccuracies [S1]. The method is part of a pipeline that minimizes reconstruction error by focusing model complexity on important voxels (those with neighbors of opposing sign) using the ground truth signs [S1].

## Sources
[S1] Deep Implicit Volume Compression, https://doi.org/10.1109/cvpr42600.2020.00137, fetched 2026-10-26
[S2] PDF Deep Learning Methods and Neural Representations for Scientific Volume ..., https://www.cs.uoregon.edu/Reports/AREA-202603-Mathai.pdf, fetched 2026-10-27 (unofficial)
