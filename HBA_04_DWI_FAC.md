# HBA_04 DWI

Human Brain Atlas Processing Tutorial [DWI Data]

## Table of Contents 

[TOC]

## About

This guide will go through the steps used to generate the DWI templates from the Human Brain Atlas project. This guide assumes :

-you have installed all the necessary software/programs
-that you're using linux or OSX as your operating system. most of the software packages used here are not compatible with Windows.

The data from the original project was upsampled to 0.50mm isotropic resolution. You can can download this dataset here (download the `demo-dwi` folder as a zip file): 

| Link to sample dataset used in this guide | [link coming soon...]                                                         |
| ----------------- |:--------------------------------------------------------------------- |

Any queries can be sent to Zoey Isherwood (zoey.isherwood@gmail.com) or Mark Schira (mark.schira@gmail.com)

**NOTE:** The method we used here to generate DWI templates upsamples the dataset as a first set. We're in the middle of optimising our methods and may opt to upsample as a final step. So this guide here serves as a record for what we original did and a new guide may surface soon... (2022/02/18 - zji).


## List of software packages needed

| Software/Programs | Website                                                               |
| ----------------- |:--------------------------------------------------------------------- |
| ITKSNAP           |[http://www.itksnap.org/pmwiki/pmwiki.php?n=Downloads.SNAP3](https://) |
| MATLAB            |[https://au.mathworks.com/products/matlab.html](https://)              |
| Python            |[https://www.python.org/](https://)                                    |
| HDBET             |[https://github.com/MIC-DKFZ/HD-BET](https://)                         | 
| FSL               |[https://fsl.fmrib.ox.ac.uk/fsl/fslwiki](https://)                     |
| Freesurfer        |[https://surfer.nmr.mgh.harvard.edu/](https://)                        |
| MRTrix3           |[https://www.mrtrix.org/](https://)                                    |
| ANTS              |[http://stnava.github.io/ANTs/](https://)                              |
| pydeface          |[https://github.com/poldracklab/pydeface](https://)                    |

## List of scripts used to process data

**NOTE:** if you download the dataset, all the necessary code is included in the zip folder.

| Scripts          | Website                                                                                 |
| ----------------- |:----------------------------------------------------------------------------------------|
| `dicm2nii.m` |[insert link here]   |
| `make-masks-hba-project.sh` |[insert link here]  |
| `preproc-hba-project.sh` | [insert link here] |

## Data summary

>:memo: Internal data summary:
>| Sequence Name          | File used for template | File used for brainmask |
>| ----------------- |:--------| :--------|
>| MP2RAGE | UNI_DEN | **INV2** |
>| DUTCH  | INV1_ND |INV1_ND |
>| FLAWS  | INV2_ND |INV2_ND |
>| T2 | T2 | T2  |
>| DTI | `mean` for alignment. Apply non-linear & linear transform to other files (`FAC`) | `mean`  |

## Things to be added by Mark Schira:

| Column 1 | Column 2 |
| -------- | -------- | 
| Processing DICOMS in MRTrix     | done    |
| Making ID files     | todo   |
|  Making Tensor Files      | todo    |
| Overlaying final FAC file on T1 template     | todo    |
| Nonlinear alignment using MRTrix     | todo    |

## Importing Philips Dicom files into MRtrix mif 

### Step 1: sorting and locating the raw DICOM data

>:memo: Internal lab info: dicoms are located here:
>```
>/mnt/lab-nas/projects/DWI-mark/
>DEV90 corresponds to HBA sub-02
>DEV98 corresponds to HBA sub-01
>The raw DICOM images were sorted into subfolders following the
>BIDS naming convention
>hence folder the raw DICOM are sorted into folders called 
>sub-01_ses-13_run-01 etc
>the raw DICOMs are located in subfolders called 201 etc.


>:memo: On Philips DICOM naming conventions. 
The MRtrix tool mrconvert can read Philips DICOM correctly importing the diffusion parameters, Philips sorts it's DICOM in folders called 201, 202, 301, 302, 303 etc.
The first digit stands for the scan number in session (hence 101 is typically the scout and often not exported), the third digit couts upwards if there are multiple types of outputs from a scan.
I.e. for a DWI scan the folder 201 could contain the raw images, and if there was for example an FAC image calculated on the scanner, this would end up in the folder 202.


### Step 2: Converting Philips DICOM to .mif
DWI DICOM data can be natively accessed using MRTRIX tools
For example calling `mrinfo 301` when located in the subfolder `.../sub-01_ses-13_run-01` will output

![](/IMG_HBA_04/gfJRRmo.png)

Here one can find to `DEV0098` the subject id used on the scanner console, the name of the exam card `dti opt32 1000 avghighb SPIR 1.25iso 4bfactavg`
the matrix size and some other parameters and most importantly the dw gradient orientations which MRtrix analysis including FSL eddy correction requires.
Importing the data to mif can be as easy as 
```
mrconvert 201 sub-01_ses-13_run-01_raw.mif
mrcat 301/ 401/ sub-01_ses-13_run-01_b0s.mif -axis 3
```
producing two files, the raw DWI data and a second file containing two B0 images with inverse blip for distortion correction. Note, the command `mrcat` invokes `mrconvert` for the raw directories 301/ and 401/ it then concatenates the two images. 
## Preprocessing
### Preprocessing Step 1: Upsampling raw data in the inplane direction

The next step is upsampling the raw data before any other preprocessing is done. This is because preprocessing such as topup or eddy correction involves resampling of the data. Any resampling involves blurring, the higher the voxel resolution, the less the blurring. Hence the very first after importing the data is

```
mrgrid -scale 2,2,1 sub-01_ses-13_run-01_raw.mif  regrid sub-01_ses-13_run-01_raw_up.mif
mrgrid -scale 2,2,1 sub-01_ses-13_run-01_b0s.mif  regrid sub-01_ses-13_run-01_b0s_up.mif
```

### Preprocession Step 2: DWIFSLPREPROC
The next step uses the very handy DWIFSLPREPROC scrpt which essentially does topup and eddy correction in one handy step

```
dwifslpreproc sub-01_ses-13_run-01_raw_up.mif sub-01_ses-13_run-01_prepro.mif -pe_dir AP -rpe_pair -se_epi sub-01_ses-13_run-01_b0s_up.mif -readout_time 0.0576 -align_seepi -eddy_options " --slm=linear "
```
The three .mif files here are: the input data, the name for the output data and the third file is the pair of B0s.

After the dwifslpreproc step it is typically a good idea to inspect the results, for example making a mean of the preprocessed dwi data.
```
mrmath sub-01_ses-13_run-01_prepro.mif mean sub-01_ses-13_run-01_mean.nii.gz -axis 3 
```
This image can then easily be inspected using ITKSnap or mrview, looking for distortions.![](/IMG_HBA_04/gRnjknh.png)



Rename the files output in `raw` with BIDS formatting. For this, use `sub-01` as the subject name, and `ses-14` as the session number. In the sample dataset, the run numbers should be runs: `run-01`, and `run-02`. The acquisition name should be: `acq-dwi-3t`.

There will be multiple files associated with each `run`. These files include: `ADC`, `FA`, `FAC_1`, `FAC_2`, `FAC_3`, and `mean`.

See below for some examples of how to rename each file:

| Original Name     | BIDS Name                                                                           |
| ----------------- |:------------------------------------------------------------------------------------|
| [insert original filename].nii.gz | sub-01_ses-14_run-01_acq-dwi-3t_ADC.nii.gz            |
| [insert original filename].nii.gz  | sub-01_ses-14_run-01_acq-dwi-3t_FAC_1.nii.gz       |
| [insert original filename].nii.gz  |   sub-01_ses-14_run-01_acq-dwi-3t_FAC_2.nii.gz  |
| [insert original filename].nii.gz  |   sub-01_ses-14_run-01_acq-dwi-3t_FAC_3.nii.gz | 
| [insert original filename].nii.gz  |   sub-01_ses-14_run-01_acq-dwi-3t_FA.nii.gz    |
| [insert original filename].nii.gz  |   sub-01_ses-14_run-01_acq-dwi-3t_mean.nii.gz    |

We were a bit old school in our approach and manually renamed each file to follow BIDS formatting. There are many more intuitive ways of doing this (e.g. using the BIDS feature in dicm2nii, naming files automatically using a script etc), but we ended up naming them manually.

Normally the next step is to deface the data so that it's anonymous. However, we don't have to do this for these files since 1) they don't contain much information other than the brain. 2) we'll be skull stripping later on, and 3) the code we typically use for defacing (`pydeface`) fails on these files. So now we'll be moving onto Preprocessing the data.

## Aligning DWI sessions into atlas space

### Step 1: Preprocess raw files using `preproc-hba-project.sh`

```bash=1
### preprocess images.
DATA_PATH=[base input path to the unzipped downloaded dataset]

CODE_DIR=${DATA_PATH}/code
OUTPUT_PATH=${DATA_PATH}/preproc

# make OUTPUT_PATH if it doesn't already exist

if [ ! -d ${OUTPUT_PATH} ]; then
  mkdir -p ${OUTPUT_PATH};
fi

#input vars:

# for preproc-hba-project.sh

NORMALISE_LUM=0 # set to 0, no need to normalise the luminance values.
UPSAMPLE_VOXEL_SIZE=0.4 #set to 0.25 for the HBA project, 0.4 for the demo script
EQDIMS_SIZE=512 # set to 1024 for the HBA project, 512 for the demo script

#list all the scans you want to process in the IMAGES variable below. #find "$(pwd)" makes it easier...
# for the HBA project, we preprocessed ALL the files we had (e.g. T1_Images, UNI_DEN, etc) just in case we wanted to analyse these images later down the track...

# but for the purposes of this demo, we'll just process the most relevant file: INV2_ND.

IMAGES=(${DATA_PATH}/raw/sub-01_ses-14_run-01_acq-dwi-3t_ADC.nii.gz
    ${DATA_PATH}/raw/sub-01_ses-14_run-01_acq-dwi-3t_FAC_1.nii.gz
    ${DATA_PATH}/raw/sub-01_ses-14_run-01_acq-dwi-3t_FAC_2.nii.gz
    ${DATA_PATH}/raw/sub-01_ses-14_run-01_acq-dwi-3t_FAC_3.nii.gz
    ${DATA_PATH}/raw/sub-01_ses-14_run-01_acq-dwi-3t_FA.nii.gz
    ${DATA_PATH}/raw/sub-01_ses-14_run-01_acq-dwi-3t_mean.nii.gz
    ${DATA_PATH}/raw/sub-01_ses-14_run-02_acq-dwi-3t_ADC.nii.gz
    ${DATA_PATH}/raw/sub-01_ses-14_run-02_acq-dwi-3t_FAC_1.nii.gz
    ${DATA_PATH}/raw/sub-01_ses-14_run-02_acq-dwi-3t_FAC_2.nii.gz
    ${DATA_PATH}/raw/sub-01_ses-14_run-02_acq-dwi-3t_FAC_3.nii.gz
    ${DATA_PATH}/raw/sub-01_ses-14_run-02_acq-dwi-3t_FA.nii.gz
    ${DATA_PATH}/raw/sub-01_ses-14_run-02_acq-dwi-3t_mean.nii.gz)
    
   for image in "${IMAGES[@]}"; do  

    bash ${CODE_DIR}/preproc-hba-project.sh \
        -i $image \
        -o $OUTPUT_PATH \
        -n $NORMALISE_LUM \
        -u $UPSAMPLE_VOXEL_SIZE \
        -e $EQDIMS_SIZE \

done
```

### Step 2: Generate automated masks for each raw file

In order to skull strip each file, we have to first generate a brainmask. To do this we use the `make-masks-hba-project.sh` script, which utilises HD-BET and ANTs' N4 bias correction. The script is pretty automated, requiring only a few input parameters, so I won't delve into exactly what it's doing here.

1. To generate brainmasks of the files generated in the last few steps, run the following section of code:

```bash=1
DATA_PATH=[base input path to the unzipped downloaded dataset]
CODE_DIR=${DATA_PATH}/code

# for make-masks-hba-project.sh

OUTPUT_PATH=${DATA_PATH}/brainmasks
INFLATE_MM=0

# make OUTPUT_PATH if it doesn't already exist

if [ ! -d ${OUTPUT_PATH} ]; then
  mkdir -p ${OUTPUT_PATH};
fi

# list all the scans you want to process in the IMAGES variable below. #find "$(pwd)" makes it easier...

IMAGES=(${DATA_PATH}/preproc/preproc-sub-01_ses-14_run-01_acq-dwi-3t_mean.nii.gz
${DATA_PATH}/preproc/preproc-sub-01_ses-14_run-02_acq-dwi-3t_mean.nii.gz)

for image in "${IMAGES[@]}"; do  

    bash ${CODE_DIR}/make-masks-hba-project.sh \
    -i $image \
    -o $OUTPUT_PATH

done

#clean up

 rm $OUTPUT_PATH/ss-*.nii.gz
 rm $OUTPUT_PATH/hd-bet-*.nii.gz
```

### Step 3: Skull strip all raw files with masks generated in the last step

```bash=1
DATA_PATH=[base input path to the unzipped downloaded dataset]
CODE_DIR=${DATA_PATH}/code

# get scans and masks and put into the following variables... need fullpath for code to work. use line below...
# find $(pwd)/*.nii.gz -maxdepth 1 -type f -not -path '*/\.*' | sort
# find $(pwd)/brain*.nii.gz -maxdepth 1 -type f -not -path '*/\.*' | sort

OUTPUT_PATH=${DATA_PATH}/brainmasks

# make OUTPUT_PATH if it doesn't already exist

if [ ! -d ${OUTPUT_PATH} ]; then
  mkdir -p ${OUTPUT_PATH};
fi

IMAGES=(${DATA_PATH}/preproc/preproc-sub-01_ses-14_run-01_acq-dwi-3t_ADC.nii.gz
    ${DATA_PATH}/preproc/preproc-sub-01_ses-14_run-01_acq-dwi-3t_FAC_1.nii.gz
    ${DATA_PATH}/preproc/preproc-sub-01_ses-14_run-01_acq-dwi-3t_FAC_2.nii.gz
    ${DATA_PATH}/preproc/preproc-sub-01_ses-14_run-01_acq-dwi-3t_FAC_3.nii.gz
    ${DATA_PATH}/preproc/preproc-sub-01_ses-14_run-01_acq-dwi-3t_FA.nii.gz
    ${DATA_PATH}/preproc/preproc-sub-01_ses-14_run-01_acq-dwi-3t_mean.nii.gz
    ${DATA_PATH}/preproc/preproc-sub-01_ses-14_run-02_acq-dwi-3t_ADC.nii.gz
    ${DATA_PATH}/preproc/preproc-sub-01_ses-14_run-02_acq-dwi-3t_FAC_1.nii.gz
    ${DATA_PATH}/preproc/preproc-sub-01_ses-14_run-02_acq-dwi-3t_FAC_2.nii.gz
    ${DATA_PATH}/preproc/preproc-sub-01_ses-14_run-02_acq-dwi-3t_FAC_3.nii.gz
    ${DATA_PATH}/preproc/preproc-sub-01_ses-14_run-02_acq-dwi-3t_FA.nii.gz
    ${DATA_PATH}/preproc/preproc-sub-01_ses-14_run-02_acq-dwi-3t_mean.nii.gz)
    
MASKS=(${DATA_PATH}/brainmasks/brainmask-preproc-sub-01_ses-14_run-01_acq-dwi-3t_mean.nii.gz
    ${DATA_PATH}/brainmasks/brainmask-preproc-sub-01_ses-14_run-01_acq-dwi-3t_mean.nii.gz
    ${DATA_PATH}/brainmasks/brainmask-preproc-sub-01_ses-14_run-01_acq-dwi-3t_mean.nii.gz
    ${DATA_PATH}/brainmasks/brainmask-preproc-sub-01_ses-14_run-01_acq-dwi-3t_mean.nii.gz
    ${DATA_PATH}/brainmasks/brainmask-preproc-sub-01_ses-14_run-01_acq-dwi-3t_mean.nii.gz
    ${DATA_PATH}/brainmasks/brainmask-preproc-sub-01_ses-14_run-01_acq-dwi-3t_mean.nii.gz
    ${DATA_PATH}/brainmasks/brainmask-preproc-sub-01_ses-14_run-02_acq-dwi-3t_mean.nii.gz
    ${DATA_PATH}/brainmasks/brainmask-preproc-sub-01_ses-14_run-02_acq-dwi-3t_mean.nii.gz
    ${DATA_PATH}/brainmasks/brainmask-preproc-sub-01_ses-14_run-02_acq-dwi-3t_mean.nii.gz
    ${DATA_PATH}/brainmasks/brainmask-preproc-sub-01_ses-14_run-02_acq-dwi-3t_mean.nii.gz
    ${DATA_PATH}/brainmasks/brainmask-preproc-sub-01_ses-14_run-02_acq-dwi-3t_mean.nii.gz
    ${DATA_PATH}/brainmasks/brainmask-preproc-sub-01_ses-14_run-02_acq-dwi-3t_mean.nii.gz)

echo  "skull stripping with indicated masks..."

counter=0

for image in "${IMAGES[@]}"; do  

    FILEPATH=$(dirname $image)
    FILENAME=$(basename $image)
    FILENAMENOEXT=${FILENAME%%.*}

    echo "ss'ing: ${image}"
    echo "brainmask: ${MASKS[$counter]}"

    ImageMath 3 ${OUTPUT_PATH}/ss-${FILENAMENOEXT}.nii.gz m $image ${MASKS[$counter]}

    # copy mask used here to final directory...

    # cp ${MASKS[$counter]} ${OUTPUT_PATH}/brainmask-${FILENAMENOEXT}.nii.gz

    ((counter=counter+1))

done
```

### Step 4: Creating an unbiased starting template using `create-template-hba-project.sh` before running the ANTs Multivariate Template script

Since we don't have a starting template to use, we need to make our own first. To do this, we'll use `FSL` to align and average each T1 scan in an unbiased way (e.g. everything won't be aligned to the first scan, rather everything will be aligned to a rough average of the scans, then linearly averaged). The script we'll use to run registration and averaging in `FSL` is called  `create-template-hba-project.sh`. If you want more detail on what exact commands are used, please refer to that script. In order to run it, see the block of code below.

For the DWI data, we only create a template using the `mean` files since the other files have basically no spatial detail to use with regular alignment scripts. The transforms used to align the `mean` files will then be applied to the corresponding other files.

```bash=1

DATA_PATH=[base input path to the unzipped downloaded dataset]
CODE_DIR=${DATA_PATH}/code

OUTPUT_PATH=${DATA_PATH}/alignment

if [ ! -d ${OUTPUT_PATH} ]; then
  mkdir -p ${OUTPUT_PATH};
fi

#output name must NOT have extension...
OUTPUT_NAME=sub-01-fsl-template-dwi-3t

IMAGES=(${DATA_PATH}/brainmasks/ss-preproc-sub-01_ses-14_run-01_acq-dwi-3t_mean.nii.gz
    ${DATA_PATH}/brainmasks/ss-preproc-sub-01_ses-14_run-02_acq-dwi-3t_mean.nii.gz)
    
IMAGES="${IMAGES[@]}"

#THE INPUT FLAG NEEDS TO BE AT THE END OTHERWISE THE CODE WON'T WORK!!!

bash ${CODE_DIR}/create-template-hba-project.sh \
    -d $OUTPUT_PATH \
    -n $OUTPUT_NAME \
    -i "${IMAGES[@]}"     
```

### Step 5. Copy `mean.nii.gz` files needed for the ANTs Multivariate Template Code

ANTs requires all the files used for the template to be in the same directory. It's not the most practical thing to do space-wise, but I like to copy all the files we'll be using into a new directory just for ANTs. So now we'll transfer over the relevant `mean.nii.gz` files... As well as the input template.

As mentioned in the previous step, we have to use the `mean.nii.gz` files to align each scan together rather than other file types (e.g. `FA`, `FA_1`, etc). The non-linear warped images and linear registration parameters used to align the `mean` files will then be applied to the other file types to align them.

```bash=1
DATA_PATH=[base input path to the unzipped downloaded dataset]
OUTPUT_PATH=${DATA_PATH}/ants-mvt/mean

if [ ! -d ${OUTPUT_PATH} ]; then
  mkdir -p ${OUTPUT_PATH};
fi

cp ${DATA_PATH}/brainmasks/ss*_mean.nii.gz $OUTPUT_PATH
cp ${DATA_PATH}/alignment/sub-01-fsl-template-dwi-3t.nii.gz $OUTPUT_PATH
```

### Step 6. Run the ANTs Multivariate Template Code

Now we're finally ready to run the ANTs Multivariate Template Code!

Run the block of code below to run ANTs. Be sure to change the variable `NUM_CORES` based on your computer specs.

Depending on your computer specs this can take a few days to run. To run 3 iterations on a 24 core computer (ramonx, UOW) it'll take ~3 days.

```bash=1
DATA_PATH=[base input path to the unzipped downloaded dataset]
CODE_DIR=${DATA_PATH}/code

FILEDIR=${DATA_PATH}/ants-mvt

TEMPLATE=${FILEDIR}/sub-01-fsl-template-dwi-3t.nii.gz

IMAGES=(${FILEDIR}/mean/ss-preproc-sub-01_ses-14_run-01_acq-dwi-3t_mean.nii.gz
        ${FILEDIR}/mean/ss-preproc-sub-01_ses-14_run-02_acq-dwi-3t_mean.nii.gz)
        
DIMS=3
GRADIENT=0.1
NUM_CORES=12 #change the number of cores based on your computer's specs
NUM_MODS=1
N4BIASCORRECT=0 # we've already done this step.
STEPSIZES=20x15x5 #from Lüsebrink, F., Sciarra, A., Mattern, H., Yakupov, R. & Speck, O. T1-weighted in vivo human whole brain MRI dataset with an ultrahigh isotropic resolution of 250 μm. Scientifc data 4, 170032, https://doi.org/10.1038/sdata.2017.32 (2017).

ITERATIONS=4 #4 for hba project. may change to 3 iterations later down the track... 2022/02/14. zji.

###############################
#
# Set number of threads
#
###############################

    ORIGINALNUMBEROFTHREADS=${ITK_GLOBAL_DEFAULT_NUMBER_OF_THREADS}
    ITK_GLOBAL_DEFAULT_NUMBER_OF_THREADS=$NUM_CORES

    export ITK_GLOBAL_DEFAULT_NUMBER_OF_THREADS

########### ants

cd $FILEDIR
outputPath=${FILEDIR}/mean/TemplateMultivariateBSplineSyN_${STEPSIZES}
mkdir $outputPath

antsMultivariateTemplateConstruction.sh \
    -d $DIMS \
    -r 1 \
    -c 0 \
    -m $STEPSIZES \
    -n $N4BIASCORRECT \
    -s CC \
    -t GR \
    -i $ITERATIONS \
    -g $GRADIENT \
    -b 1 \
    -o ${outputPath}/T_ \
    -y 0 \
    -z $TEMPLATE \
     ${IMAGES[@]}

###############################
#
# Restore original number of threads
#
###############################

    ITK_GLOBAL_DEFAULT_NUMBER_OF_THREADS=$ORIGINALNUMBEROFTHREADS

    export ITK_GLOBAL_DEFAULT_NUMBER_OF_THREADS
    
# copy template to main dir

cp ${outputPath}/T_template0.nii.gz ${FILEDIR}/ants-mvt-template_mean.nii.gz
```

### Step 7. Looking at/navigating the data from ANTs.

Once the script has completed, you can inspect your data using ITKSNAP. You'll notice in the `${DATA_PATH}/ants-mvt` folder a new folder called `${DATA_PATH}/ants-mvt/TemplateMultivariateBSplineSyN_${STEPSIZES}` depending on the step sizes you used. Within this folder, you'll see some other folders with `GR_` as the prefix. These folders are output after each iteration. If you want to compare the output template after each iteration you can open the `T_template0.nii.gz` file within each `GR_` folder and compare it to the final template `${DATA_PATH}/ants-mvt/TemplateMultivariateBSplineSyN_${STEPSIZES}/T_template0.nii.gz`.

The other files you may see have the suffixes `_Warp.nii.gz`, `InverseWarp.nii.gz`, and `Warp.nii.gz`. These are the non-linear alignment files output from ANTs which were used to warp the files to the input template. The number preceding each suffix indicates the file number it corresponds to (e.g. scans 1 to 2 will correspond to 0 to 1).

Files with the suffix `_WarpedToTemplate.nii.gz` are the output of each scan being warped to the template. Again the number preceding the suffix indicates the file number, so if you open each one they should all be aligned.

As noted above, the output template we're most interested in has the file name `T_template0.nii.gz`. See below for an example screenshot of this file:

![](/IMG_HBA_04/WtjBffA.png)

### Step 8. Applying warps/alignment parameters from Step 6 to other scans (`FAC_1`,`FAC_2`,`FAC_3`)

```bash=1
# user input---------------------------------------------------------------------

DATA_PATH=[base input path to the unzipped downloaded dataset]

FILEDIR=${DATA_PATH}/ants-mvt/FAC
MEAN_TEMPLATE_DIR=${DATA_PATH}/ants-mvt/mean/TemplateMultivariateBSplineSyN_20x15x5
DIM=3

if [ ! -d ${FILEDIR} ]; then
  mkdir -p ${FILEDIR};
fi

# ls $(pwd)/ss*.nii.gz
IMAGES=(${DATA_PATH}/brainmasks/ss-*FAC*.nii.gz)

IMAGES=(${DATA_PATH}/brainmasks/ss-preproc-sub-01_ses-14_run-01_acq-dwi-3t_FAC_1.nii.gz
    ${DATA_PATH}/brainmasks/ss-preproc-sub-01_ses-14_run-01_acq-dwi-3t_FAC_2.nii.gz
    ${DATA_PATH}/brainmasks/ss-preproc-sub-01_ses-14_run-01_acq-dwi-3t_FAC_3.nii.gz
    ${DATA_PATH}/brainmasks/ss-preproc-sub-01_ses-14_run-02_acq-dwi-3t_FAC_1.nii.gz
    ${DATA_PATH}/brainmasks/ss-preproc-sub-01_ses-14_run-02_acq-dwi-3t_FAC_2.nii.gz
    ${DATA_PATH}/brainmasks/ss-preproc-sub-01_ses-14_run-02_acq-dwi-3t_FAC_3.nii.gz)

#ls $(pwd)/*Warp.nii.gz, repeat each warp image 3 times (for each FAC image)
WARP_IMAGES=(${MEAN_TEMPLATE_DIR}/T_ss-preproc-sub-01_ses-14_run-01_acq-dwi-3t_mean0Warp.nii.gz
    ${MEAN_TEMPLATE_DIR}/T_ss-preproc-sub-01_ses-14_run-01_acq-dwi-3t_mean0Warp.nii.gz
    ${MEAN_TEMPLATE_DIR}/T_ss-preproc-sub-01_ses-14_run-01_acq-dwi-3t_mean0Warp.nii.gz
    ${MEAN_TEMPLATE_DIR}/T_ss-preproc-sub-01_ses-14_run-02_acq-dwi-3t_mean1Warp.nii.gz
    ${MEAN_TEMPLATE_DIR}/T_ss-preproc-sub-01_ses-14_run-02_acq-dwi-3t_mean1Warp.nii.gz
    ${MEAN_TEMPLATE_DIR}/T_ss-preproc-sub-01_ses-14_run-02_acq-dwi-3t_mean1Warp.nii.gz)

# ls $(pwd)/*txt, repeat each txt transform 3 times (for each FAC image)
WARP_AFFINE_MATRICES=(${MEAN_TEMPLATE_DIR}/T_ss-preproc-sub-01_ses-14_run-01_acq-dwi-3t_mean0Affine.txt
    ${MEAN_TEMPLATE_DIR}/T_ss-preproc-sub-01_ses-14_run-01_acq-dwi-3t_mean0Affine.txt
    ${MEAN_TEMPLATE_DIR}/T_ss-preproc-sub-01_ses-14_run-01_acq-dwi-3t_mean0Affine.txt
    ${MEAN_TEMPLATE_DIR}/T_ss-preproc-sub-01_ses-14_run-02_acq-dwi-3t_mean1Affine.txt
    ${MEAN_TEMPLATE_DIR}/T_ss-preproc-sub-01_ses-14_run-02_acq-dwi-3t_mean1Affine.txt
    ${MEAN_TEMPLATE_DIR}/T_ss-preproc-sub-01_ses-14_run-02_acq-dwi-3t_mean1Affine.txt)

# ls $(pwd)/T_template0.nii.gz
TEMPLATE=(${MEAN_TEMPLATE_DIR}/T_template0.nii.gz)

# depending on the number of scans you have, repeat numbers from 0 to X 3 times (for each FAC image)
TEMPLATE_NUM=(0
    0
    0
    1
    1
    1)

# start processing-----------------------------------------------------------------

counter=0

for image in "${IMAGES[@]}"; do  

    ### get file names/paths

    FILEPATH=$(dirname ${IMAGES[$counter]})
    FILENAME=$(basename ${IMAGES[$counter]})
    FILENAMENOEXT=${FILENAME%%.*}

    WarpImageMultiTransform \
        $DIM \
        ${IMAGES[$counter]} \
        ${FILEDIR}/T_template0${FILENAMENOEXT}${TEMPLATE_NUM[$counter]}WarpedToTemplate.nii.gz \
        -R ${TEMPLATE} \
        ${WARP_IMAGES[$counter]} \
        ${WARP_AFFINE_MATRICES[$counter]}

    ((counter=counter+1))

done

# now average the each warped FAC file (FAC_1, FAC_2, FAC_3) across runs using the same method as the ANTs Multivariate Template script

for counter in 1 2 3; do  # 3 FAC directions...

TEMPLATE_FOLDER="${FILEDIR}/FAC_${counter}"
#TEMPLATE_FOLDER="${FILEDIR}"

if [ ! -d ${TEMPLATE_FOLDER} ]; then
  mkdir -p ${TEMPLATE_FOLDER};
fi

#note: for the indicated template below, ensure that this is the input template that ants
# generates at the beginning of the antsMultivariateTemplate.sh script... I usually generate
# this input template before running the antsMVT script, so transfer this file to the template_folder,
# and rename it "T_template0.nii.gz"... 

# copy template file from the mean multivariate template script output.

TEMPLATE="${TEMPLATE_FOLDER}/T_template0.nii.gz"

if [ ! -f $TEMPLATE ]; then
     cp ${DATA_PATH}/ants-mvt/mean/sub-01-fsl-template-dwi-3t.nii.gz ${TEMPLATE_FOLDER}/T_template0.nii.gz;
fi

# copy all other necessary files to the template folder

TEMPLATE_AFFINE="${TEMPLATE_FOLDER}/T_template0Affine.txt"
if [ ! -f $TEMPLATE_AFFINE ]; then
     cp ${MEAN_TEMPLATE_DIR}/T_template0Affine.txt ${TEMPLATE_FOLDER}/T_template0Affine.txt;
fi

TEMPLATE_WARP="${TEMPLATE_FOLDER}/T_template0Warp.nii.gz"
if [ ! -f $TEMPLATE_WARP ]; then
     cp ${MEAN_TEMPLATE_DIR}/T_template0Warp.nii.gz ${TEMPLATE_FOLDER}/T_template0Warp.nii.gz;
fi

TEMPLATE_WARP_LOG="${TEMPLATE_FOLDER}/T_templatewarplog.txt"
if [ ! -f $TEMPLATE_WARP_LOG ]; then
     cp ${MEAN_TEMPLATE_DIR}/T_templatewarplog.txt ${TEMPLATE_FOLDER}/T_templatewarplog.txt;
fi

# move relevant warped FAC files

mv ${FILEDIR}/*FAC_${counter}*WarpedToTemplate.nii.gz ${TEMPLATE_FOLDER}

# start the averaging procedure:

DIMS=3
GRADIENT=0.1
NUM_CORES=12 #change depending on the specs of your computer; ramonx 12

###############################
#
# Set number of threads
#
###############################

ORIGINALNUMBEROFTHREADS=${ITK_GLOBAL_DEFAULT_NUMBER_OF_THREADS}
ITK_GLOBAL_DEFAULT_NUMBER_OF_THREADS=$NUM_CORES

export ITK_GLOBAL_DEFAULT_NUMBER_OF_THREADS


########### ants

# local declaration of values
dim=$DIMS
template=$TEMPLATE
templatename="${TEMPLATE_FOLDER}/T_template"
outputname="${TEMPLATE_FOLDER}/T_"
gradientstep=-$GRADIENT
whichtemplate=0

echo "shapeupdatetotemplate()"

# debug only
# echo $dim
# echo ${template}
# echo ${templatename}
# echo ${outputname}
# echo ${outputname}*WarpedToTemplate.nii*
# echo ${gradientstep}

# We find the average warp to the template and apply its inverse to the template image
# This keeps the template shape stable over multiple iterations of template building

echo
echo "--------------------------------------------------------------------------------------"
echo " shapeupdatetotemplate---voxel-wise averaging of the warped images to the current template"
echo "   ${ANTSPATH}/AverageImages $dim ${template} 1 ${templatename}${whichtemplate}*WarpedToTemplate.nii.gz    "
echo "--------------------------------------------------------------------------------------"
${ANTSPATH}/AverageImages $dim ${template} 1 ${templatename}${whichtemplate}*WarpedToTemplate.nii.gz

if [[ $whichtemplate -eq 0 ]] ;
  then
    echo
    echo "--------------------------------------------------------------------------------------"
    echo " shapeupdatetotemplate---voxel-wise averaging of the inverse warp fields (from subject to template)"
    echo "   ${ANTSPATH}/AverageImages $dim ${templatename}${whichtemplate}warp.nii.gz 0 ${outputname}*Warp.nii.gz"
    echo "--------------------------------------------------------------------------------------"

    ${ANTSPATH}/AverageImages $dim ${templatename}${whichtemplate}warp.nii.gz 0 ${outputname}*Warp.nii.gz

    echo
    echo "--------------------------------------------------------------------------------------"
    echo " shapeupdatetotemplate---scale the averaged inverse warp field by the gradient step"
    echo "   ${ANTSPATH}/MultiplyImages $dim ${templatename}${whichtemplate}warp.nii.gz ${gradientstep} ${templatename}${whichtemplate}warp.nii.gz"
    echo "--------------------------------------------------------------------------------------"

    ${ANTSPATH}/MultiplyImages $dim ${templatename}${whichtemplate}warp.nii.gz ${gradientstep} ${templatename}${whichtemplate}warp.nii.gz

    echo
    echo "--------------------------------------------------------------------------------------"
    echo " shapeupdatetotemplate---average the affine transforms (template <-> subject)"
    echo "                      ---transform the inverse field by the resulting average affine transform"
    echo "   ${ANTSPATH}/AverageAffineTransform  ${dim} ${templatename}0Affine.txt ${outputname}*Affine.txt"
    echo "   ${ANTSPATH}/WarpImageMultiTransform ${dim} ${templatename}0warp.nii.gz ${templatename}0warp.nii.gz -i  ${templatename}0Affine.txt -R ${template}"
    echo "--------------------------------------------------------------------------------------"

    ${ANTSPATH}/AverageAffineTransform ${dim} ${templatename}0Affine.txt ${outputname}*Affine.txt
    ${ANTSPATH}/WarpImageMultiTransform ${dim} ${templatename}0warp.nii.gz ${templatename}0warp.nii.gz -i  ${templatename}0Affine.txt -R ${template}

    ${ANTSPATH}/MeasureMinMaxMean ${dim} ${templatename}0warp.nii.gz ${templatename}warplog.txt 1
  fi

echo "--------------------------------------------------------------------------------------"
echo " shapeupdatetotemplate---warp each template by the resulting transforms"
echo "   ${ANTSPATH}/WarpImageMultiTransform ${dim} ${template} ${template} -i ${templatename}0Affine.txt ${templatename}0warp.nii.gz                                         ${templatename}0warp.nii.gz ${templatename}0warp.nii.gz ${templatename}0warp.nii.gz -R ${template}"
echo "--------------------------------------------------------------------------------------"
${ANTSPATH}/WarpImageMultiTransform ${dim} ${template} ${template} -i ${templatename}0Affine.txt ${templatename}0warp.nii.gz ${templatename}0warp.nii.gz           ${templatename}0warp.nii.gz ${templatename}0warp.nii.gz -R ${template}
    
done

# copy final templates to main directory

cp ${FILEDIR}/FAC_1/T_template0.nii.gz ${FILEDIR}/ants-mvt-template_FAC_1.nii.gz
cp ${FILEDIR}/FAC_2/T_template0.nii.gz ${FILEDIR}/ants-mvt-template_FAC_2.nii.gz
cp ${FILEDIR}/FAC_3/T_template0.nii.gz ${FILEDIR}/ants-mvt-template_FAC_3.nii.gz

# combine the 3 FAC files together to make 1 FAC file.

IMAGES=(${FILEDIR}/ants-mvt-template_FAC_1.nii.gz
        ${FILEDIR}/ants-mvt-template_FAC_2.nii.gz
        ${FILEDIR}/ants-mvt-template_FAC_3.nii.gz)
time_spacing=3
time_origin=0
NDIMS=4
OUTPUT_NAME=ants-mvt-template_FAC.nii.gz

ImageMath \
    $NDIMS \
    ${FILEDIR}/$OUTPUT_NAME \
    TimeSeriesAssemble $time_spacing $time_origin \
    ${IMAGES[@]}

#now you're done!
```

### Step 9. Aligning the DWI output to our canonical 0.25mm T2 template.

We now want to get our DWI data in Template space. To do this we will align our `mean` template to the 0.25 mm T2 template. We do this since they're closer in contrast than the T1 template. We will align the two images using ANTs non-linear alignment, then apply the transform to the `FAC` template.

1. Before running the alignment in ANTs, we will do a rough alignment in ITK-SNAP and save the transformation matrix as a `txt` file to be used in ANTs as a starting point.

    You can open the relevant files using the ITKSNAP GUI, otherwise you can use the code block immediately below:

    ```bash=1
    T2_TEMPLATE=${DATA_PATH}/template/sub-01_t2_ACPC.nii.gz
    MEAN_TEMPLATE=${DATA_PATH}/ants-mvt/mean/ants-mvt-template_mean.nii.gz

    itksnap -g $T2_TEMPLATE -o $MEAN_TEMPLATE
    ```
    
    When the GUI opens up, in the menu bar click `Tools` then `Registration`, then the `Manual` tab in the Registration toolbar.
    
  ![](/IMG_HBA_04/DFAzv4B.png)
  
  
Manually align until both images roughly match. I try to roughly align both images so that the ACPC line is aligned.
    
Now you can click the `Automatic` option in the Registration Toolbar. You can use the default settings `Tranformation Model: Rigid, Image Similarity Metric: Mutual Information, Coarsest Level: 16x, Finest Level 8x`, then click `Run Registration`.

Once the registration is complete, save the transformation matrix with the filename `ants-mvt-template_mean-rigid-16x8x.txt` in the `${DATA_PATH}/ants-mvt/mean` folder.

![](/IMG_HBA_04/paxyG8l.png)

2. Now you're ready to start the ANTs non-linear alignment. Run the code block below to do this.

```bash=1
DATA_PATH=[base input path to the unzipped downloaded dataset]

FILEDIR=${DATA_PATH}

if [ ! -d ${FILEDIR} ]; then
  mkdir -p ${FILEDIR};
fi

# Register the $fixed and $moving images
# with initial alignment of the centers
# of intensity followed by the following
# three stages:
#   rigid -> affine
#
# modified from klein et al 2009

nCores=12 #12 #select 1 for macbook pro, 12 for ramonx

INFLATE_VX=5

DIMS=3 #dimension of your data

prefix=reg-

outputdir=${DATA_PATH}/ants-mvt

PRECISION=f  #d - double, f - float

fixed=${DATA_PATH}/template/sub-01_t2_ACPC.nii.gz

moving=(${DATA_PATH}/ants-mvt/mean/ants-mvt-template_mean.nii.gz)

moving_transforms=(${DATA_PATH}/ants-mvt/mean/ants-mvt-template_mean-rigid-16x8x.txt)
    

###############################
#
# No more user input past this point...
#
###############################

    echo "Reference Image: $fixed"

    echo "Image(s) to be moved: ${moving[@]}"

###############################
#
# Set number of threads
#
###############################

    ORIGINALNUMBEROFTHREADS=${ITK_GLOBAL_DEFAULT_NUMBER_OF_THREADS}
    ITK_GLOBAL_DEFAULT_NUMBER_OF_THREADS=$nCores

    export ITK_GLOBAL_DEFAULT_NUMBER_OF_THREADS

###############################
#
# Run ANTs command
#
###############################

counter=0

for image in "${moving[@]}"; do

    FILEPATH=$(dirname $image)
    FILENAME=$(basename $image)
    FILENAMENOEXT=${FILENAME%%.*}
    
    regprefix=${outputdir}/${prefix}${FILENAMENOEXT}-

    outputname=${outputdir}/${prefix}${FILENAME}

    # rigid + affine + nonlinear --------------------------------------

    antsRegistration \
        --dimensionality 3 \
        --output ${regprefix} \
        --use-histogram-matching 1 \
        --initial-moving-transform ${moving_transforms[$counter]} \
        --transform Rigid[0.1] \
        --metric MI[${fixed},${image},1,32,Regular,0.25] \
        --convergence 1000x500x250x100 \
        --smoothing-sigmas 3x2x1x0 \
        --shrink-factors 8x4x2x1 \
        --transform Affine[0.1] \
        --metric MI[${fixed},${image},1,32,Regular,0.25] \
        --convergence 1000x500x250x100 \
        --smoothing-sigmas 3x2x1x0 \
        --shrink-factors 8x4x2x1 \
        --transform BSplineSyN[0.1,26,0,3] \
        --metric CC[${fixed},${image},1,4] \
        --convergence 100x70x50x20 \
        --smoothing-sigmas 3x2x1x0 \
        --shrink-factors 6x4x2x1 \
        --float 1 \
        --verbose

    # Apply the resulting transforms (generic
    # affine + B-spline SyN) to the
    # moving labels.

    antsApplyTransforms \
        --dimensionality 3 \
        --input ${image} \
        --reference-image ${fixed} \
        --output ${outputname} \
        --n BSpline[5] \
        --transform ${regprefix}1Warp.nii.gz \
        --transform ${regprefix}0GenericAffine.mat \
        --default-value 0 \
        --float 1 \
        --verbose

    ((counter=counter+1))

done
```

3. Now apply transform from Step 2 to the FAC files.

To do this, run the code block below:

```bash=1
DATA_PATH=[base input path to the unzipped downloaded dataset]

images=(${DATA_PATH}/ants-mvt/FAC/ants-mvt-template_FAC_1.nii.gz
        ${DATA_PATH}/ants-mvt/FAC/ants-mvt-template_FAC_2.nii.gz
        ${DATA_PATH}/ants-mvt/FAC/ants-mvt-template_FAC_3.nii.gz)

fixed=${DATA_PATH}/template/sub-01_t2_ACPC.nii.gz

# warpimage=/mnt/lab-nas/projects/human-brain-atlas/ForPrint/sub-01/data/reg-ants-mvt-template_mean-1InverseWarp.nii.gz
warpimage=${DATA_PATH}/ants-mvt/reg-ants-mvt-template_mean-1Warp.nii.gz
affinematrix=${DATA_PATH}/ants-mvt/reg-ants-mvt-template_mean-0GenericAffine.mat

for image in "${images[@]}"; do  

    FILEPATH=$(dirname $image)
    FILENAME=$(basename $image)
    FILENAMENOEXT=${FILENAME%%.*}

    outputname=${DATA_PATH}/ants-mvt/reg-${FILENAME}

    antsApplyTransforms \
        --dimensionality 3 \
        --input ${image} \
        --reference-image ${fixed} \
        --output ${outputname} \
        --n BSpline[5] \
        --transform ${warpimage} \
        --transform ${affinematrix} \
        --default-value 0 \
        --float 1 \
        --verbose

done

## combine 3 FAC files into one file:
    
    IMAGES=(${DATA_PATH}/ants-mvt/reg-ants-mvt-template_FAC_1.nii.gz
            ${DATA_PATH}/ants-mvt/reg-ants-mvt-template_FAC_2.nii.gz
            ${DATA_PATH}/ants-mvt/reg-ants-mvt-template_FAC_3.nii.gz)
    time_spacing=3
    time_origin=0
    NDIMS=4
    OUTPUT_NAME=reg-ants-mvt-template_FAC.nii.gz

    ImageMath \
        $NDIMS \
        ${DATA_PATH}/ants-mvt/$OUTPUT_NAME \
        TimeSeriesAssemble $time_spacing $time_origin \
        ${IMAGES[@]}        
```

[input image of final transformed scan here]
