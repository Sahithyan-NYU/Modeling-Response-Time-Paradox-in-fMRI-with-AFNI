# Modeling-Response-Time-Paradox-in-fMRI-with-AFNI


This repository contains the scripts for processing and analyzing functional MRI (fMRI) data.

Specifically, modeling the Response Time Paradox according to Mumford, J.A., Bissett, P.G., Jones, H.M. et al. The response time paradox in functional magnetic resonance imaging analyses. Nat Hum Behav 8, 349–360 (2024). https://doi.org/10.1038/s41562-023-01760-0 

The pipeline consists of three main steps:

1. **Step 1: Marrying Task Onset and Reaction Time Regressors**
2. **Step 2: Creating mean-centred RT modulated regressor using AM2**
3. **Step 3: Multiple Regression Analysis**

Each step is executed with specific commands and processing steps using AFNI tools such as `3dDeconvolve` and `1dMarry`.

## Prerequisites

- AFNI (Analysis of Functional NeuroImages) tools installed.
- A functional MRI dataset for each subject.
- Subject-specific regressor files for task onsets and reaction times.
- Template brain mask and motion parameters files.
- A Unix/Linux environment with shell scripting capabilities.

## File Structure

```
/media/cogemolab/home2/PCR
├── data/
│   └── individual/
│       └── PCR<subject_id>/
│           ├── func/
│           └── regressors/
│               └── activation/
│                   └── mainTask/
│                       ├── M_TaskOnset_PCR<subj>.txt
│                       ├── M_RT_Regressor_PCR<subj>.txt
│                       ├── M_PositiveOnset_PCR<subj>.txt
│                       ├── M_NeutralOnset_PCR<subj>.txt
│                       ├── M_NegativeOnset_PCR<subj>.txt
│                       └── M_ErrorTaskOnset_PCR<subj>.txt
│           └── activation/
│               └── wholeBrain/
│                   └── mainTask/
│                       ├── AM2/
│                       ├── RT_Regressed/
```

## Steps

### Step 1: Marrying Task Onset and Reaction Time Regressors

This step combines the task onset and reaction time files for each subject. The script `marry_regressors.csh` takes the task onset file and reaction time regressor file for each subject and marries them into a single file using the `1dMarry` command.

#### Command:
```bash
#!/bin/csh

set proj_path=/media/cogemolab/home2/PCR

foreach subj ( `seq 501 1 550` )
    echo $subj
    cd $proj_path/data/individual/PCR"$subj"/regressors/activation/mainTask
    1dMarry M_TaskOnset_PCR"$subj".txt M_RT_Regressor_PCR"$subj".txt > M_TaskOnset_RT_married_PCR"$subj"
end
```

- **Input**: Task onset and reaction time regressor files for each subject.
- **Output**: Merged file of task onset and reaction time regressor (`M_TaskOnset_RT_married_PCR<subj>`).

### Step 2: Creating mean-centred RT modulated regressor using AM2

We forcestop the 3dDeconvolve, to extract the mean-centred RT modulated regressor using AM2, so that it can be used in the next step.
We are performing this step only to create mean-centred RT modulated regressor. 

#### Command:
```bash
#!/bin/csh

set proj_path=/media/cogemolab/home2/PCR
set temp_path=/media/cogemolab/home2/templates
set subj=$1
set reg_path=$proj_path/data/individual/PCR"$subj"/regressors/activation/mainTask

mkdir -p $proj_path/data/individual/PCR"$subj"/activation/wholeBrain/mainTask/AM2
cd $proj_path/data/individual/PCR"$subj"/activation/wholeBrain/mainTask/AM2

3dDeconvolve \
    -overwrite \
    -input $proj_path/data/individual/PCR"$subj"/func/mainTask/PCR"$subj"_M_TR_MNI_3mm_SI.nii.gz \
    -mask $temp_path/MNI152_T1_3mm_brain_GM_02182017.nii.gz \
    -concat '1D: 0 186 372 558 744 930' \
    -polort A \
    -nobout \
    -noFDR \
    -num_stimts 1 \
    -local_times \
    -stim_times_AM2 1 $reg_path/M_TaskOnset_RT_married_PCR"$subj" 'GAM(8.6,0.547,1)' -stim_label 1 M_RT_Married_AM2 \
    -x1D ./"PCR"$subj"_M_RT_married_AM2_CueDeconvTaskMR.x1D" \
    -xsave \
    -xjpeg ./"PCR"$subj"_M_RT_married_AM2_CueDeconvTaskMR.png" \
    -x1D_stop
```

- **Input**: Functional MRI data, task regressors (including the merged task and reaction time regressor), brain mask, motion parameters.
- **Output**: mean-centred RT modulated regressor
  
### Step 3: Multiple Regression Analysis 

This step performs a multiple regression analysis using the `3dDeconvolve` tool, modeling the brain activity response to different task conditions (positive, neutral, negative, task error) and including a mean-centred reaction time regressor along with motion parameters. 

#### Command:
```bash
#!/bin/csh

set proj_path=/media/cogemolab/home2/PCR
set temp_path=/media/cogemolab/home2/templates
set subj=$1
set reg_path=$proj_path/data/individual/PCR"$subj"/regressors/activation/mainTask

mkdir -p $proj_path/data/individual/PCR"$subj"/activation/wholeBrain/mainTask/RT_Regressed
cd $proj_path/data/individual/PCR"$subj"/activation/wholeBrain/mainTask/RT_Regressed

echo "Subject $subj ... running multiple regression using 3dDeconvolve:"

3dDeconvolve \
    -overwrite \
    -input $proj_path/data/individual/PCR"$subj"/func/mainTask/PCR"$subj"_M_TR_MNI_3mm_SI.nii.gz \
    -mask $temp_path/MNI152_T1_3mm_brain_GM_02182017.nii.gz \
    -concat '1D: 0 186 372 558 744 930' \
    -polort A \
    -nobout \
    -noFDR \
    -num_stimts 5 \
    -local_times \
    -stim_times 1 $reg_path/M_PositiveOnset_PCR"$subj".txt 'GAM(8.6,0.547,1.5)' -stim_label 1 Positive \
    -stim_times 2 $reg_path/M_NeutralOnset_PCR"$subj".txt 'GAM(8.6,0.547,1.5)' -stim_label 2 Neutral \
    -stim_times 3 $reg_path/M_NegativeOnset_PCR"$subj".txt 'GAM(8.6,0.547,1.5)' -stim_label 3 Negative \
    -stim_times 4 $reg_path/M_ErrorTaskOnset_PCR"$subj".txt 'GAM(8.6,0.547,1.5)' -stim_label 4 TaskError \
    -stim_file 5 $proj_path/data/individual/PCR"$subj"/activation/wholeBrain/mainTask/AM2/RT_regressor_mean_centered_PCR"$subj".1D -stim_label 5 M_RT \
    -ortvec $proj_path/data/individual/PCR"$subj"/func/mainTask/PCR"$subj"_M_MotionPar.txt'[1..6]' 'MotionParam' \
    -ortvec $proj_path/data/individual/PCR"$subj"/func/mainTask/PCR"$subj"_M_MotionPar_derv.1D'[1..6]' 'MotionParamDerv' \
    -censor $proj_path/data/individual/PCR"$subj"/func/mainTask/PCR"$subj"_M_censor.1D \
    -cbucket ./"PCR"$subj"_M_TR_MNI_3mm_SI_censor_CueDeconvTaskMR_betas_RT.nii.gz" \
    -x1D ./"PCR"$subj"_M_TR_MNI_3mm_SI_censor_CueDeconvTaskMR_RT.x1D" \
    -x1D_uncensored ./"PCR"$subj"_M_TR_MNI_3mm_SI_Uncensor_CueDeconvTaskMR_RT.x1D" \
    -xsave \
    -xjpeg ./"PCR"$subj"_M_censor_CueDeconvTaskMR_RT.png" \
    -fitts ./"PCR"$subj"_M_TR_MNI_3mm_SI_censor_CueDeconvTaskMR_fitts_RT.nii.gz" \
    -errts ./"PCR"$subj"_M_TR_MNI_3mm_SI_censor_CueDeconvTaskMR_errts_RT.nii.gz" \
    -tout \
    -bucket ./"PCR"$subj"_M_TR_MNI_3mm_SI_censor_CueDeconvTaskMR_RT.nii.gz"
```

- **Input**: Functional MRI data, task regressors (positive, neutral, negative, task error), mean-centred reaction time regressor, motion parameters, and censoring file.
- **Output**: Regression results (betas, fitted and error time series, design-matrix, etc.).

## Notes

- This pipeline assumes that the subject data is in a consistent format for

 all subjects (same naming conventions, directory structure, etc.).
- It’s crucial to update the file paths in the scripts to match the locations of your data and templates.
- The scripts require a shell environment (e.g., `csh`) and may require administrative permissions depending on your system configuration.

## Authors

- Sahithyan (Emotion and Cognition lab, IISc; Hatley Lab, NYU)
