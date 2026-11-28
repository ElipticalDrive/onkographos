# Standard and Baseline Fourier Encoding

Also called: Fixed Sine/Cosine Positional Encoding, Standard Fourier Positional Encoding

## Overview
This family provides deterministic, fixed-frequency mappings of low-dimensional coordinates to a higher-dimensional space using interleaved sine and cosine functions. It solves the problem of neural networks struggling to represent high-frequency details by explicitly injecting band-limited frequency information into the input representation [S1, 18385]. It differs from its siblings by using static, non-learned frequencies (typically exponential powers of two) rather than randomized features or learned covariance matrices, serving as a static feature initialization layer [18385].

## Shared principles
All methods in this family map an input coordinate vector $\mathbf{x} \in \mathbb{R}^D$ to a feature vector $\mathbf{z} \in \mathbb{R}^{2LD}$ using a set of $L$ scalar frequency values $\{\gamma_0, \gamma_1, \dots, \gamma_{L-1}\}$. The mapping is defined by evaluating sine and cosine functions at scaled versions of the input coordinates.

The general form for a single dimension $x_d$ and frequency index $l$ is:
$$ u_{l,d} = \sin(\gamma_l \cdot x_d) $$
$$ v_{l,d} = \cos(\gamma_l \cdot x_d) $$

The final feature vector is constructed by concatenating these components. The specific ordering and frequency generation rules vary by child method, but the core mechanism remains the deterministic evaluation of trigonometric basis functions.

## Subtopics

### [Fourier Feature Networks](2765-fourier-feature-networks.md)
Fourier Feature Networks (FFN) address spectral bias by replacing the network's internal featurizer with a fixed, high-dimensional Fourier embedding [2765]. Unlike standard MLPs that must learn high-frequency kernels, FFNs use a random frequency matrix $\mathbf{B} \in \mathbb{R}^{d \times B}$ to project inputs into a space where high-frequency variations become linear [2765]. The embedding is computed as $\Gamma(\mathbf{x}) = [\cos(2\pi \mathbf{B}\mathbf{x}), \sin(2\pi \mathbf{B}\mathbf{x})]$ [2765]. The matrix $\mathbf{B}$ is typically initialized with elements drawn from a Gaussian distribution $\mathcal{N}(0, \sigma^2)$ or a uniform distribution [2765]. This method allows standard MLPs with ReLU activations to predict complex signals effectively [2765].

### [Standard Fourier Positional Encoding](18385-standard-fourier-positional-encoding.md)
Standard Fourier Positional Encoding maps coordinates using fixed, deterministic frequencies rather than randomized ones [18385]. It uses $L$ frequency bands, typically generated as $\gamma_l = 2^l$ for $l=0,\dots,L-1$ [18385]. The output vector concatenates sine and cosine terms for each dimension and frequency band: $[\sin(\gamma_0 \mathbf{x}), \cos(\gamma_0 \mathbf{x}), \dots, \sin(\gamma_{L-1} \mathbf{x}), \cos(\gamma_{L-1} \mathbf{x})]$ [18385]. In reference implementations, $L=10$ is common [18385]. This serves as a static feature initialization layer for subsequent neural processing [18385].

### Fixed Sine/Cosine Positional Encoding
A baseline variant using fixed sine/cosine positional encoding for 3D Gaussian centers, often used in appearance modeling ablations [S1]. On the Brandenburg Gate scene, this method achieves PSNR 18.66, SSIM 0.763, LPIPS 0.347 [S1]. On the Trevi Fountain scene, it achieves PSNR 19.91, SSIM 0.741, LPIPS 0.305 [S1]. It is compared against affine color transformations and learned hash encodings (SWAG), with SWAG outperforming this baseline in handling in-the-wild appearance variations [S1].

## Background
The concept of using Fourier features to mitigate spectral bias in neural networks was introduced by [2765]. The specific variant using fixed exponential frequencies is widely adopted as a standard baseline in implicit neural representations [18385]. The "Fixed Sine/Cosine" variant is cited in recent work on splatting in the wild, where it serves as a comparison point for appearance modeling techniques [S1].

## Sources
[S1] SWAG: Splatting in the Wild images with Appearance-conditioned Gaussians, https://doi.org/10.48550/arXiv.2403.10427, fetched 2026-10-29
