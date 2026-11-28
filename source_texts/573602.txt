# Volumetric_Light_Shadow
Volumetric rendering of 3D Object with volumetric lighting and shadows 

Steps taken to complete openGL coding for volumetric lighting and shadow –

1. Setup offscreen rendering using FBO (both light buffer and eye buffer)
2. Load Volume Data from .raw file
3. Calculate Shadow matrix by multiplying the model view and projections matrices of light with bias matrix.
4. Calculate halfway vector using light direction and view direction. Slice the Volume data in the direction which is halfway between the view and light vectors
5. After binding the FBO, first clear the light buffer with white color and then eye buffer with black color.
6. Bind the volume VAO and then run a loop for the total number of slices. In each iteration, first render the slice in the eye buffer but bind the light buffer as the texture. Next, render the slice in the light buffer:
7. Based on the viewer direction, swap the blend function in the eye buffer rendering. 
8. For light buffer, blend the slices using the conventional over blending method.
9. Finally, unbind the FBO and restore the default draw buffer. Next, set the viewport to the entire screen and then render the eye buffer on screen using a shader:
