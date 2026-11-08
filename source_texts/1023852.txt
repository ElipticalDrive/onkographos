# Dual Autodiff

This repository contains the dual_autodiff package for the use and manipulation of Dual numbers.

Dual numbers have particular use in geomtry, computation and automatic differentiation. 

# How to use this repository


## python implementation dual_autodiff

The source code for the implementation of the dual numbers package in python is located in the dual_autodiff folder. This may be downloaded via cloning the repository to your local machine and using 

    pip install . 

From inside the repository folder (not within the dual_autodiff folder)

Once installed the dual_autodiff package may be imported with 

import dual_autodiff


The functionality of the package is contained within the Dual class which may be accessed via dual_autodiff.Dual(Real, Dual). More examples of the use of the package may be found in the tutorial notebook "dual_autodiff.ipynb" or in the package documenation.

Alternativly one my way download the python wheels of the dual autodiff_package located in the /dist folder. 

## Cython implementation dual_autodiff_x


The cythonised version of the package is located in the dual_autodiff_x folder. This may be installed via running

    pip install . 

from inside the dual_autodiff_x folder. Alternatively dual_autodiff_x may be installed using the wheels located in the dual_autodiff_x/wheelhouse folder. These are for the linux system which may be accessed either via the linux machine or via the docker container provided.

## Documentation 

Normally documentation for the package would be housed on read the docs, however as this cant be done due to the assesed nature of the project documentation may be built locally by  
1) cloning the repo to local machine, 
2) Navigating to the docs folder 
3) installing dependencies from the requirements.txt folder
4) running "make html" command.

This should generate html files in the build folder. Open the index.html file within the html folder in your browser to view it.

## Notebooks

The repository also contains the following notebooks to answer questions. Before running the notebooks ensure to have installed dual_autodiff and dual_autodiff_x via one of the methods mentioned above.

-  Q5_differentation.ipynb. This analyses differentiation a function using dual numbers vs using the numerical derivitive and aswers the question posed in part 5. Additional commentary on this is in the report

- Q9_CythonVsPython.ipynb contains the analyses of speed between the python and cython implementation answering part 9 of the project. Once again more details on this are contained within the report. 

- dual_autodiff.ipynb. This serves as an example notebook of how to use the package. It also is rendered in the documentation.


## Dependecy handling 

Dependencies for the dual_python module are located in the pyproject.toml file and are installed when dual_autodiff is installed.

To build documentation then packages in the docs_requirements.txt file in the docs folder should be pip installed.

To run notebooks dual_autodiff.ipynb, Q5_differentaition.ipynb and Q9_cythonVsPython.ipynb then packages in the requiremenst.txt should be installed as well as the dual_autodiff and dual_autodiff_x moduel.

## Docker

A docker image has been created to run this project in a linux environment with python 3.10. 

The docker image includes

- python (dual_autodiff) package installed
- Cython (dual_autodiff_x) package installed
- All Jupyter notebooks: Q5_differentation.ipynb, Q9_CythonVsPython.ipynb and dual_autofdiff.ipynb

To build the Docker image 

1) Ensure docker is installed on your machine, this may be found at https://www.docker.com/

2) After cloning the repository build the image using 
    docker build `-t dual_autodiff .`

    This will build docker container called dual_autodiff

Using the docker container

Once built one may use the docker container in one of 2 ways


1) run the conatiner:  

    `docker run -it -p 8888:8888 my-jupyter /bin/bash`

This gives you access to the linux environment inluding python and both dual_autodiff and dual_autodiff_x packages.

To use the jupityer notebooks in the docker

2) Once the docker image has been built run the following command:

    `docker run -p 8888:8888 dual_autodiff_image`

2) run http://localhost:8888/ in your chosen browser

3) Enter the token displayed in the terminal to the prompt to access the notebooks


    
## Tests

The test folder houses all tests for the dual_autodiff module. Once cloned these test may be ran executing the command 
    `pytest -s tests /*`
