# This is a rough guide how to make a combined DEC_T1 image

* We should have the goal to develop this guide to generate a detailed documentation how the DWI data from the CBD clinical trial was processed

First follow the batman tutorial for preprocessing DWI data, the steps described here assume that you have just finished dwifslpreproc and the output is DWI_preproc.mif

### Part one: calulating tensor and FAC image. 
`dwi2tensor DWI_preproc.mif DWI_tensor.mif`    
Next calculate the FAC (fractional anisotropy color) .  The step involves two processes, specifically, first calculating the direction from the tensor and the second taking the absolute value of that and writing this into a nii.gz file, which can be viewed in ITKSnap  
`tensor2metric DWI_tensor.mif -vec - | mrcalc - -abs DWI_FAC.nii.gz`  

### Part two: performing sperical deconvolution:  
calculate a response function: we want wm.txt  
`dwi2response dhollander DWI_preproc.mif wm.txt gm.txt csf.txt`  

Do the deconvolution (we want an FOD file i.e. fibre density distribution)   
`dwi2fod msmt_csd  DWI_preproc.mif wm.txt  DWI_FOD.mif`   

A FOD file has about 46 volumes it can be inspected using `mrview` For visual inspection using ITKSnap but also aligment and such, extract the first volume of the FOD file  
`mrconvert DWI_FOD.mif -coord 3 0 -axes 0,1,2 DWI_FOD_firstVol.nii.gz`





