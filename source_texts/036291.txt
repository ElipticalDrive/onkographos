This was built and tested with clang 20.

Clone the repo: `git clone https://github.com/Shibodd/volume_path_tracer --recurse-submodules`

Inside the repository folder:

To build, `cmake -S . -B build -DCMAKE_BUILD_TYPE=Release`

To run, `build/vpt scenes/YOURSCENE.json YOUROUTPUTFILE.png`. Note that quitting with CTRL+C will not save the image, you have to quit with ESC or the window close button.

To visualize a single ray for debugging, `build/visualize_ray scenes/YOURSCENE.json`

All customizable parameters are in the scene json file.

The provided volumes folder is to be put in the repository root.
