# TRACKS

Monte Carlo particles' tracks generation and reconstruction with multiple scattering and noise. Developed by Luca Quaglia and Mattia Ivaldi - Torino, 2018.  

TRACKS requires a working version of ROOT on your device. It has an easy 3-steps usage:
1. open and edit the file _detector_info.txt_ with the format  
`layer_name width radius thickness theta_rms`

2. open and edit the tracks.C macro with your preferences  
`gROOT->ProcessLine(“tracks_gen(a,b,c,d,e,f,g,h)”)` where  
bool a = verbose mode ON/OFF  
bool b = print and save plots ON/OFF  
bool c = multiple scattering ON/OFF    
bool d = noise ON/OFF  
int e = 5 custom z, 10 custom multiplicity, 15 both custom  
int f = # of collisions performed  
double g = custom vertex z  
double h = custom event multiplicity  
`gROOT->ProcessLine(“tracks_reco(a,b,c,d,e,f)”)` where  
bool a = verbose mode ON/OFF  
bool b = print and save plots ON/OFF  
double c = smearing parameter on z  
double d = smearing parameter on phi  
double e = ambiguity check amplitude  
int f = ambiguity check width

3. open a ROOT session and interpret root [0] .x tracks.C

By default the lines 30 and 31 of tracks.C are commented, if you want to perform a performances study you just need to uncomment these lines and edit the spit_perform.C and cluster_study.C macros.

The generated data will be stored in the _gen.root_ file, the reconstruction plots will be stored in the _reco.root_ file, the performances plots will be stored in the _perform.root_file. All the plots are anyway saved in .eps format in the tracksplot folder.
