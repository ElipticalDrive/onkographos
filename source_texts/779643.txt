# [A Novel Fourier Feature Network for Solving Partial Differential Equations](https://github.com/SummerLoveRain/FENs)

Building on the foundation of single-hidden-layer neural networks, Fourier Feature Networks (FENs) are proposed, which incorporate Fourier features using $\cos$, $\sin$, or a combination of both. Similar to Extreme Learning Machines (ELMs), FENs employ a single-hidden-layer architecture to generate a set of basis functions. The target function is then approximated as a linear combination of these basis functions, with the coefficients determined using the least squares method. However, unlike ELMs, which often rely on affine transformations to improve representational power, FENs can achieve high-precision solutions without requiring such transformations on the input variables. To evaluate the representational capacity of these networks, we search for an optimal scaling factor within a predefined range for the randomly initialized and fixed weights and biases. By adjusting this scaling factor, we ensure a fair comparison between FENs and ELMs using various activation functions, such as $\text{sigmoid}$, $\tanh$, and $\text{swish}$. Our numerical experiments demonstrate that FENs consistently achieve higher accuracy than ELMs.

For more information, please refer to the following: (https://doi.org/10.1016/j.cnsns.2025.109274)

## Citation

	@article{YANG2025109274,
		title = {A Novel Fourier Feature Network for Solving Partial Differential Equations},
		journal = {Communications in Nonlinear Science and Numerical Simulation},
		pages = {109274},
		year = {2025},
		issn = {1007-5704},
		doi = {https://doi.org/10.1016/j.cnsns.2025.109274},
		url = {https://www.sciencedirect.com/science/article/pii/S1007570425006847},
		author = {Qihong Yang and Zhijie Su and Yangtao Deng and Qiaolin He}
	}
