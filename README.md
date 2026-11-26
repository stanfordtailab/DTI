# Diffusion MRI Preprocessing Pipeline

[![DOI](https://zenodo.org/badge/DOI/10.5281/zenodo.17503171.svg)](https://doi.org/10.5281/zenodo.17503171)

Developed at the **Stanford Translational AI Lab (STAI)**  
**Lead Developer:** Mohammad H. Abbasi, Gustavo Chau

**Supervisor:** Ehsan Adeli  

This repository contains Python code to perform preprocessing of diffusion MRI (dMRI) data.  
At present, the only distortion correction supported is **top-up correction**, which requires an additional reversed polarity acquisition.  

---

## Prerequisites

- [FSL](https://fsl.fmrib.ox.ac.uk/fsl/)  
- [ANTs](https://github.com/ANTsX/ANTs)  
- [DIPY](https://docs.dipy.org/stable/user_guide/installation)  

---

## Configuration

Paths and file naming are controlled by `config.py`.  

All scripts work on a **subject-by-subject** basis.  

Configuration file for neuroimaging pipeline

Contains settings for:
1. Common configurations (paths, logging, etc.)
2. Topup and Eddy correction
3. Skull stripping
4. DTI fit
5. Registration

---

## Processing Steps

### 1. Topup Correction, `b0_correction.py`
Correct susceptibility-induced distortions using FSL TOPUP (Andersson et al., 2003).  
This step estimates a field map from images with opposite phase-encoding directions and applies it to unwarp distorted diffusion volumes.  

**Reference:** Andersson, J.L.R., Skare, S., & Ashburner, J. (2003). *NeuroImage, 20(2), 870–888.*

- https://pubmed.ncbi.nlm.nih.gov/14568458/

---

### 2. Eddy Current and Motion Correction, `process_eddy.py`
Correct distortions caused by eddy currents and subject motion using FSL EDDY (Andersson & Sotiropoulos, 2016).  
Includes slice-to-volume correction and outlier replacement.  

Steps performed:
1. Load and merge diffusion-weighted images (AP/PA or reversed polarity)
2. Prepare acquisition parameters and B0 indices
3. Run FSL EDDY with slice-to-volume correction (optional)
4. Generate QC images comparing original vs corrected volumes

**Reference:** Andersson, J.L.R., & Sotiropoulos, S.N. (2016). *NeuroImage, 125, 1063–1078.*

- https://pubmed.ncbi.nlm.nih.gov/26481672/

---

### 3. Brain Extraction, `brain_extraction.py`
Removes non-brain tissue from diffusion MRI data using FSL BET 
(Smith, 2002). This step performs skull stripping and generates 
quality control (QC) images and summary files to validate 
the extracted brain.

Steps performed:
1. Apply FSL BET to remove non-brain tissue
2. Generate brain mask and masked output
3. Compute brain volume (ml) and validate against the expected range
4. Create QC images (raw, brain-extracted, raw+mask, differences)
5. Write QC summary (CSV) for downstream review

**Reference:** Smith, S.M. (2002). *Human Brain Mapping, 17(3), 143–155.*

- https://pubmed.ncbi.nlm.nih.gov/12391568/

---

### 4. Merging of Acquisitions, `reg_within_fsl.py`

Aligns and merges multiple diffusion MRI runs using FSL FLIRT in case a session is broken down into multiple acquisitions.
Rigid transformations are estimated between B0 reference images,
applied to corresponding diffusion volumes, and propagated to
b-vectors to ensure orientation consistency across runs.
Steps performed:
1. Register B0 images between runs using FLIRT rigid-body transform
2. Apply transforms to diffusion volumes (DWI)
3. Rotate b-vectors using the computed transformation matrix
4. Merge registered DWI volumes, b-vectors, and b-values
5. Save combined outputs for downstream processing

---

### 5. Registration to MNI Space, `run_reg_mni.py`
Registers diffusion images to the MNI152 template using a
two-step approach with FSL FLIRT (rigid + affine). Corresponding
b-vectors are rotated to preserve orientation
consistency. The pipeline also applies the transforms to the diffusion
volumes and masks, and prepares final outputs for downstream analysis.

Steps performed:
1. Register B0 to MNI (rigid, 6 DOF), save the matrix and registered B0
2. Refine B0→MNI with affine (12 DOF), save matrix, and registered B0
3. Apply rigid and then affine transforms to DWI
4. Rotate b-vectors using the previously computed transformations
5. Copy b-values (bval_final.bval) for consistency
6. Register mask with nearest-neighbour interpolation using the previously computed transformations 

---

### 6. Diffusion Tensor Model Fitting, `run_dtifit_dipy.py`
Fits a diffusion tensor model using DIPY (Garyfallidis et al., 2014).
This script takes preprocessed diffusion MRI data (DWI, bvec, bval, mask)
and outputs standard DTI-derived measures including fractional anisotropy (FA),
mean diffusivity (MD), radial diffusivity (RD), and axial diffusivity (AD).
Eigenvectors of the tensor are also saved for visualization.
Steps performed:
1. Load diffusion MRI data, b-values, b-vectors, and brain mask
2. Construct gradient table using DIPY
3. Fit the diffusion tensor model voxel-wise
4. Extract tensor components (Dxx, Dxy, Dxz, Dyy, Dyz, Dzz)
5. Save DTI-derived tensor components and maps: FA, MD, RD, AD, and eigenvectors (V1, V2, V3)
6. Generate QC images showing a color FA map

**Reference:** 
- Basser, P.J., Mattiello, J., & LeBihan, D. (1994). MR diffusion tensor spectroscopy 
  and imaging. Biophysical Journal, 66(1), 259–267.

  - https://pubmed.ncbi.nlm.nih.gov/8130344/
  
- Garyfallidis, E., Brett, M., Correia, M.M., Williams, G.B., & Nimmo-Smith, I. (2014). 
  DIPY, a library for the analysis of diffusion MRI data. Frontiers in Neuroinformatics, 8, 8. 
  
  - https://pmc.ncbi.nlm.nih.gov/articles/PMC3931231/

---

### 7. Quality Control of Final Outputs, `run_final_qc.py`
Performs automated, session-aware QC checks on diffusion pipeline outputs.
This script validates file existence and assesses registration quality using
overlap metrics (Dice) between warped images and the MNI template. Results are
written to per-session CSV reports for downstream review.

What it checks:
1) File existence validation (key outputs, matrices, QC images)
2) Registration to MNI (rigid + affine), Dice coefficient vs. MNI template
3) Within-session registration (if NUM_SCANS_PER_SESSION > 1), Dice on b0-to-b0
4) Session discovery and per-session reporting
5) Structured logging to file + console

Outputs (CSV per subject/session):
- QC/file_existance.csv
- QC/within_subject_registraction_qc.csv              (only if NUM_SCANS_PER_SESSION > 1)
- QC/mni_registraction_qc.csv

Notes:
- Uses thresholds for PASS/WARNING based on Dice (configurable in code).
- Relies on TEMPLATE_PATH and derivative folders from config.py.
- Designed to run after registration steps have completed.

**Reference:** 
Zou, K.H., Warfield, S.K., Bharatha, A., Tempany, C.M., Kaus, M.R., Haker, S.J., 
Wells, W.M., Jolesz, F.A., & Kikinis, R. (2004), 11(2), 178–189.

- https://pubmed.ncbi.nlm.nih.gov/14974593/

---

### 8. Generation of HTML QC Reports, `dti_qc.py`
Generates automated, subject-level QC reports that summarize and visualize the
outputs of the DTI preprocessing pipeline. This script *reads existing* QC CSVs
and QC images from each step and compiles them into a clickable HTML report 
along with CSV/JSON summaries. If FA/MD maps are available, it also computes 
basic statistics.

Summarized in the report:
- Raw vs. corrected B0 (Topup), QC images (before/after)
- Eddy-corrected vs. uncorrected volumes, QC images
- Brain extraction evaluation, mask overlays + brain volume (mL)
- FA and color-FA maps, quick-look thumbnails and stats
- Registration metrics, Dice coefficients for within-session & MNI steps

Inputs (read-only):
- Existing QC CSVs (e.g., `file_existance.csv`,
  `within_subject_registraction_qc.csv`, `mni_registraction_qc.csv`)
- QC images from each step (Topup, Skull stripping, Eddy, DTI fit)
- Optional DTI maps: `dipy_fa.nii.gz`, `dipy_md.nii.gz`
- Paths and flags from `config.py` (e.g., `OUTPUT_DIR`, `NUM_SCANS_PER_SESSION`)

Outputs:
- Per-subject JSON:  `<QC>/<subject_id>/<subject_id>_qc_results.json`
- Per-subject CSV:   `<QC>/<subject_id>/<subject_id>_qc_summary.csv`
- Per-subject HTML:  `<QC>/<subject_id>/<subject_id>_report.html`
- Aggregated CSV:    `<QC>/all_subjects_summary.csv`
- Aggregated HTML:   `<QC>/DTI_QC_Summary.html`

Behavior & notes:
- Session-aware: if session directories exist (e.g., YYYY-MM-DD), they are 
  automatically detected and linked.
- Read-only summarization; heavy QC computations (e.g., Dice) are performed 
  in earlier scripts.
- If `nilearn` is not available, visualization sections are safely skipped.
- Uses the non-interactive Matplotlib backend (`Agg`) to run in headless 
  environments.

**Reference:** 
Dice for overlap validation: Zou, K.H., et al. (2004). Academic Radiology, 11(2), 178–189.

 - https://pubmed.ncbi.nlm.nih.gov/14974593/


---

### 9. Utility Functions, `utilities.py`

Provides supporting functions used across the diffusion MRI preprocessing pipeline.


Key functionalities:

- Session Detection, automatically identifies session folders (e.g., YYYY-MM-DD).

- File Search & Matching, robust pattern matching to locate NIfTI, bvec/bval, and transform files even when filenames differ slightly.

- NIfTI Handling, inspects image dimensions and trims odd dimensions to ensure compatibility with FSL/ANTs.

- QC Image Generation, creates slice-wise PNG snapshots for quick quality checks.

- Logging Utilities, initializes console and file logging with safe fallbacks if write access is restricted.

These utilities are imported by other scripts (e.g., b0_correction.py, process_eddy.py, brain_extraction.py) to ensure consistency and avoid code duplication.

---

### Authors:
Mohammad H Abbasi (mabbasi [at] stanford.edu)

Gustavo Chau (gchau [at] stanford.edu)

---


Stanford University, STAI Lab (https://stai.stanford.edu)

Created: 2025

Version: 1.0.0


---
## Quick Start

### 1) Edit configuration
- Update paths and file patterns in **`config.py`** to point to your dataset.  
- Set `NUM_SCANS_PER_SESSION` according to your acquisitions.  

---

### 2) Prepare subject list
Create a text file named **`subject_list.txt`** with one subject ID per line, e.g.:

- sub-0001
- sub-0002
- sub-0003
  
### 3) Run preprocessing

Submit array jobs (replace N with the number of subjects in your list)

```bash
sbatch --array=1-N run_b0_correction.sh
```

### Pipeline order
- Pipeline order (per subject)
- python b0_correction.py <subject_id>
- python process_eddy.py <subject_id>
- python brain_extraction.py <subject_id>
- python reg_within_fsl.py <subject_id>
- python run_reg_mni.py <subject_id>
- python run_dtifit_dipy.py <subject_id>
- python run_final_qc.py <subject_id>
- python dti_qc.py <subject_id>

### Check outputs
Check outputs

- Preprocessed data → <OUTPUT_DIR> (set in config.py)
- QC results → <OUTPUT_DIR>/QC/
- Open <OUTPUT_DIR>/QC/DTI_QC_Summary.html for an overview across all subjects
  
---

## 📘 Citation
*(short version)*  
> Abbasi, M.H., & Chau, G. & Adeli, E (2025). *DTI Processing Pipeline.* Zenodo.  
> DOI: [10.5281/zenodo.17503171](https://doi.org/10.5281/zenodo.17503171)

---

## 🧾 How to Cite This Work
If you use this pipeline in your research, please cite it as follows:

> **Abbasi, M.H.**, **Chau, G.**, & **Adeli, E** (2025). *DTI Processing Pipeline: A reproducible framework for diffusion MRI preprocessing, tensor fitting, and quality control using MRtrix3 and FSL.*  
> Stanford Translational AI Lab, Stanford University. Zenodo.  
> [https://doi.org/10.5281/zenodo.17503171](https://doi.org/10.5281/zenodo.17503171)

```bibtex
@software{abbasi2025dti,
  author       = {MohammadHassan Abbasi,  Ehsan Adeli, and Gustavo Chau},
  title        = {DTI Processing Pipeline: A reproducible framework for diffusion MRI preprocessing and quality control},
  year         = {2025},
  institution  = {Stanford Translational AI Lab, Stanford University},
  publisher    = {Zenodo},
  doi          = {10.5281/zenodo.17503171},
  url          = {https://doi.org/10.5281/zenodo.17503171}
}
```

## License

This project is licensed under the MIT License; see the [LICENSE](LICENSE) file for details.
