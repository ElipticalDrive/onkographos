## Spherical Volume Renderer: Srend   
2022nov07 (v 20221107)   Ted Wetherbee   

Contact: ted@fdltcc.edu  OR  ted@tedwetherbee.org

Available:  http://tedwetherbee.org/srend  and  https://github.com/tedwetherbee/srend

Much earlier versions are available at `http://srend.nongnu.org`  

**********************************************************************
## srend.zip contains the following

#### MANUALS
```
srend_man.pdf
vrd_man.pdf
```

#### SOURCE in src/
```
Srend files:   srend.F90   Fortran Srend code
               srendc.c    C code for sleep routines, and LodePNG
                           written by Lode Vandevenne.  LodePNG  
                           source and other code is available:
                             https://lodev.org/lodepng/
                             https://github.com/lvandeve/lodepng
               srend.H     header file for C++
               
Test script:   t.sh        preps and runs test codes for Srend
                           and Vrd
                           
Test codes:    t_*         Test codes in Fortran, C, C++, and Python
                           to exercise Srend features
                          
Vrd:           vrd.F90     Source code for Vrd
Vrd scripts    vrd.script* Vrd scripts exercises Vrd features
Vrd data       vrd*bob*    Fortran programs to generate BOB data
                           for Vrd scripts
```                 
**********************************************************************
## Quickstart to test Srend and Vrd with test t_* programs
```
1. cd src
2. edit t.sh to select/add compilers with MPI library
3. chmod 755 t.sh
4. ./t.sh   # run the test programs

Images and movies are created in the current directory, then moved
to the t_result_imagery/ directory when done.  If all went well, 
Errors = 0.  63MB of images and movies should be created.  If ffmpeg
was not in the path, movies won't exist.

It is helpful to have several CPU cores for MPI tests, but not required.
```
