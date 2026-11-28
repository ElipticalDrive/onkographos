# offset-cone-beam-CT-reconstruction
Code to suport: Allan H, Doherty A, Navarrete-León C, et al. "Multi-contrast X-ray microtomography of human lung specimens with an extended ﬁeld-of-view". *Medical Physics*. 2026.

DOI link: https://doi.org/10.1002/mp.70335

Scripts are provided to simulate central-slice sinograms of an offset cone-beam geometry, and then reconstruct using the demonstrated redundancy weighting scheme. While this code was written in the context of phase-sensitive imaging techniques, the weighting scheme applies equally to conventional tomography data. Two versions are supplied: one which uses fan-beam geometry definitions to allow a demonstration of the method using just a CPU (no GPU), and another using cone-beam geometry definitions (requires GPU) which is more readily adapted to a user's own cone-beam CT data. The fan-beam demonstration should run in just a few seconds on a basic PC. Code is designed to demonstrate the principle, rather than be optimised for fast data processing.

Uses standard python packages in addition to the Astra toolbox (https://astra-toolbox.com/index.html).

![A simple comparison of a computational phantom reconstructed with and without redundancy weights.](weightedReconSim_comparison.png)
Shepp-Logan computational phantom reconstructed with and without redundancy weights (post-convolution weighting, central-slice cone-beam geometry).
