# Guide to neurocommand on NIASRA

## basic usage
* connect using ssh:
`ssh -J schiralab@hpc.niasra.uow.edu.au schiralab@niasra-node4`    
* start neurocommand by typing
`conda activate neurocommand`   

* list all available modules `ml avail`   
       
## Freesurfer
i.e. `ml freesurfer/7.3.2` 
### note on subject output SUBJECTS_DIR
* Freesurfer will work right out of the box on NIASRA, because we set an important variable in .bashrc
* by default freesurfer writes subject output into the folder set in $SUBJECTS_DIR. In the container this variable point to a directory INSIDE the container, hence it tries write the recon output into the container, this would result in recon-all to fail. So one has to set the *EXPORT SINGULARITYENV_SUBJECTS_DIR=~/WHEREEVER_IS_A_BETTER_PLACE*
*This is already defined in the `.bashrc` so it should work right out of the box
* if you are unhappy with the subject output directory
~/freesurfer/subjects
you can set a different one yourself by exporting the `EXPORT SINGULARITYENV_SUBJECTS_DIR= ....`
* the prefix SINGUARITYENV is needed to overwrite a variable that is already known inside the container.

## MRtrix
i.e. `ml mrtrix3/3.0.2`  
only the 3.0.2 module has currently cuda support, so `dwifslpreproc` will run very quick. 
