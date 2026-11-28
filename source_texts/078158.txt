# Volume Data Visualization

This software allows for the 3D visualization of volumetric datasets. The current implementation makes use of raycasting in order to render the data using various ray functions such as maximum and average intensity projection, first-hit ray casting and compositing. More features such as a custom transfer function, and support for other ray functions are in progress. The .GIF below shows a demonstration of the current functionality of the software. The dataset used in the demonstration is the CT scan of a <i>Pseudotherium argentinus</i> gathered from [1].

<img src="img/output.gif" alt="Demo .gif showing the user interface and camera controls" width="100%"/>

## Dependencies

- [ImGUI](https://github.com/ocornut/imgui/), [ImGradientHDR](https://github.com/effekseer/ImGradientHDR) and [GLFW](https://github.com/glfw/glfw) for the UI and input controls
- OpenGL Extension Wrangler Library (or GLEW, for extended OpenGL 3D rendering functionality, [link](https://glew.sourceforge.net/))
- OpenGL Mathematics (or glm, mathematics library used for linear algebra functions, [link](https://github.com/g-truc/glm))

## References:

[1] Dr. Rachel Wallace, Ricardo Martínez and Timothy Rowe, 2019, "Pseudotherium argentinus" (On-line), Digital Morphology. Accessed October 10, 2022 at http://digimorph.org/specimens/Pseudotherium_argentinus/.
