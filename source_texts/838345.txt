An implementation of a denoising algorithm. Math and further explanation inside .ipynb.

Restoring images is central across computer vision and imaging tasks, among others. Many denoising algorithms, such as Gaussian blurring, lose important details like edges or textures from the original image. Here, we compute total variation as a sum of the magnitudes of the derivatives in the horizontal and vertical, decoupling the directions from each other. This allows the algorithm to smooth homogeneous regions while preserving sharp edges. Optimization of the separable problems is done using the Alternating Direction Method of Multipliers (ADMM) method.


Created for the UMass graduate COSC590 course Applied Numerical Optimization, in collaboration with Vaibhav Shah and Frank Chiu.
