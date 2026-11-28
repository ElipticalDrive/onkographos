# Wavelet Kernel Selection

Also called: Haar/5x3/9x7/(6, 6) Kernel Comparison, Wavelet Kernel Parameters

## Overview
Wavelet Kernel Selection determines the specific filter coefficients used in the separable 1D filter bank of a wavelet transform for volume data compression [S1]. It solves the trade-off between computational reversibility and energy compaction efficiency, selecting between integer-based kernels for lossless coding and floating-point kernels for lossy coding [S1]. It is distinguished from sibling methods by its focus on the specific algebraic properties of the basis functions (e.g., vanishing moments, orthogonality) rather than the hierarchical decomposition structure itself [S1].

## How it works
### Inputs and Outputs
- **Input:** Compression mode (lossless or lossy/near-lossless), target bit-rate, and volume data characteristics (e.g., axial resolution, inter-slice correlation) [S1].
- **Output:** A selected kernel configuration defined by parameters $MK, PK, U, G, L, H, k, c_{P,i,k}, c_{U,i,k}$ [S1].

### Data Structures
The kernel is defined by a set of lifting and scaling parameters [S1]:
- $MK$: Number of lifting steps for the prediction filter.
- $PK$: Number of lifting steps for the update filter.
- $U$: Number of upsampling factors.
- $G$: Gain factor.
- $L$: Low-pass filter coefficient scaling.
- $H$: High-pass filter coefficient scaling.
- $k$: Index offsets for the filter taps.
- $c_{P,i,k}$: Coefficients for the prediction lifting steps.
- $c_{U,i,k}$: Coefficients for the update lifting steps.

### Selection Steps
1. Determine the compression mode (lossless vs. lossy/near-lossless) based on application requirements [S1].
2. Evaluate volume characteristics, particularly axial resolution and inter-slice correlation, to assess expected bit-rate gains from directional decomposition [S1].
3. Select the kernel configuration that matches the mode and data properties from the predefined set [S1].
4. Apply the selected kernel's parameters to the separable 1D filter bank for decomposition [S1].

### Kernel Configurations
The following configurations are standard [S1]:

1. **Haar Kernel** (Lossless)
   - $MK=1, PK=1, U=1, G=1, L=1, H=1, k=0$
   - $c_{P,i,k} = [-1, 1]$
   - $c_{U,i,k} = [0, 0.5]$
   - Characteristics: Simplest orthogonal wavelet with the shortest support [S1]. Less efficient than 5x3 in lossless scenarios due to shorter support length [S1].

2. **5x3 Kernel** (Lossless)
   - $MK=1, PK=1, U=1, G=1, L=1, H=1, k=0$
   - $c_{P,i,k} = [-1, 1]$
   - $c_{U,i,k} = [0.5, 0.25]$
   - Characteristics: Biorthogonal CDF 5/3 wavelet with rational coefficients where dividends are powers of two, enabling perfect reversibility without rounding errors [S1]. Performs better than Haar in lossless compression due to longer support length [S1]. Selected by the JPEG committee for lossless compression in JPEG 2000 [S1].

3. **9x7 Kernel** (Lossy/Near-Lossless)
   - $MK=2, PK=1, U=1, G=0, L=0.812893066, H=1, k=[0, -1]$
   - $c_{P,i,k} = [1.230174105, -0.052980119, -0.882911076]$
   - $c_{U,i,k} = [0.443506852]$
   - Characteristics: Biorthogonal, nearly orthogonal variation of the CDF cubic B-spline construction with four vanishing moments per wavelet filter [S1]. Offers improved energy compaction for lossy compression compared to 5x3 [S1].

4. **(6,6) Kernel** (Lossy/Near-Lossless)
   - $MK=1, PK=3, U=3, G=1, L=1.306139349, H=1.140657705, k=[0, -1, -2, 2, -3, 3]$
   - $c_{P,i,k} = [150/256, 150/512, -25/256, -25/512, 3/256, 3/512]$
   - $c_{U,i,k}$: Not given in the sources.
   - Characteristics: Interpolation wavelet offering higher energy compaction efficiency than 9x7 for some datasets [S1].

### Edge Cases and Pitfalls
- **Axial Resolution:** Volumes with low axial resolution (e.g., CT6, MRI3) have less inter-slice correlation, resulting in smaller bit-rate gains from axial decomposition [S1].
- **Precision:** 5x3 uses rational coefficients with powers-of-two dividends to ensure perfect reversibility without rounding errors [S1].
- **Complexity:** (6,6) and 9x7 have higher computational complexity than Haar and 5x3 due to more lifting steps and longer filter support [S1].

## Subtopics

### JPEG 2000 5x3 Wavelet Kernel
A biorthogonal wavelet kernel constructed from the CDF 5/3, featuring rational coefficients with dividends as powers of two, enabling perfect reversibility without loss of performance [S1]. Its synthesis scaling function is a linear B-spline [S1]. It is selected by the JPEG committee for lossless compression in JPEG 2000 [S1].

## Operating characteristics
- **Time Complexity:** Higher for 9x7 and (6,6) due to more lifting steps and longer filter support compared to Haar and 5x3 [S1].
- **Memory Complexity:** Depends on the number of coefficients stored. 5x3 and Haar are integer-based, potentially allowing more compact storage if quantization is applied, but 5x3 is preferred for lossless [S1].
- **Performance:**
  - **Lossless:** 5x3 > Haar [S1].
  - **Lossy/Near-Lossless:** (6,6) $\approx$ 9x7 > 5x3 > Haar. (6,6) outperforms 9x7 for 7 out of 11 tested images [S1]. BD-PSNR improvements up to 2 dB for 9x7/(6,6) vs 5x3 at 0.5–3.0 bpp [S1].
- **Strengths:** 5x3 offers perfect reversibility. 9x7/(6,6) offer superior energy compaction for lossy coding [S1].
- **Limitations:** 9x7/(6,6) are irreversible. Haar is inefficient for lossless compared to 5x3 [S1].

## Background
Wavelet kernel selection is a critical component of wavelet-based volumetric compression, particularly in medical imaging (e.g., JP3D, JPEG 2000) [S1]. The 5x3 and 9x7 kernels were selected by the JPEG committee for lossless and lossy compression, respectively, based on exhaustive studies of energy compaction and reversibility [S1]. The (6,6) kernel is an alternative interpolation wavelet proposed in literature for improved energy compaction in specific datasets [S1].

## Sources
[S1] Wavelet based volumetric medical image compression, https://doi.org/10.1016/j.image.2014.12.007, fetched 2026-10-26
