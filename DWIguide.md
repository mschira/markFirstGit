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

The last step would be to extract the DEC from thiss FOD image.
In principle that command is `fod2dec` but there are a few other things, such as having a T1.nii.gz ready and aligned. Also, while in principle you can make a decent DEC from a single DWI scan, we ultimately want to derive it from BOTH DWI scans. To do that we need calculate an FOD from each, align the two FODs and then average. 


### difference between DEC and FAC 
 Almost the same but sorta different.  FAC is derived from a tensor and DEC can be derived from anything, possebly a tensor, but here in this case from the FOD.




