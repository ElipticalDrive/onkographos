# PerlinNoiseGeneration
2D Terrain Generation Algorithms created for the **Procedural Content Generation** (PCG) course.

Perlin noise is a texture generation algorithm, which use same sized particles to increase controllability and realism of the output.

The algorithm’s work could be divided into three steps:
1.	Assign grid with random height values. This process is called White noise and it assigns values from 0 to 1 for each element of the grid (array items).
2.	Generate specified number of octaves. Each octave is a unique Smooth noise for the grid, which builds curver with specific wavelength and frequency.  That curve is build by sampling initial height of the grid and interpolating between this and obtained values.
3.	Glue octaves together. Add weighted by the specified amplitude values of corresponding smooth arrays into the final height map. 
Set of smooth noises create cloudy effect of the height map, which gives it more realism.
