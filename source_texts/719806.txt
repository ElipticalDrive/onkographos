# path_tracer.js
Path tracer written in JS. Uses blinn-phong for BRDF and russian roulette importance sampling.

## Run Instructions
Open the index.html file in any web browser. If any JSON files are not loading, you may have to open the project via VS Code and run the program via the Live Server extension. If running correctly, you should see individual pixels being shaded.

## Features
- 100 samples per pixel
- Russian roulette
- 2 area lights
- Some refraction
- Specular BRDF

## Screenshot
![screenshot](https://github.com/swaggimus-grime/path_tracer.js/blob/925edbefed093cc278ef58182914aff89eb882b7/screenshot.png)
