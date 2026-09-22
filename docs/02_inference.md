# Inference Guide

## Overview

The inference pipeline takes a patient DICOM folder (CT + RTSTRUCT) and produces:
- A predicted dose **NIfTI** (`.nii.gz`) anchored to the native CT geometry
- A predicted dose **RTDOSE DICOM** (`.dcm`) importable into any clinical TPS (Eclipse, Monaco, RayStation)
- A linked **dummy RTPLAN DICOM** (`RP.dummy_plan.dcm`) if no clinical plan is present — required for TPS RTDOSE import

All three outputs are saved back into the patient's DICOM series folder.

---

## 1. Requirements

| Item | Details |
|---|---|
| Python environment | `dose_env` (activate before running) |
| Script | `utils/inference_pipeline.py` |
| Config | `config/config.yml` |
| Model weights | `model/<model_name>.pth` |
| Patient data | DICOM folder containing **CT + RTSTRUCT** (RTPLAN optional) |

> Run all commands from the `01 ICON/` directory.

---

## 2. Single-Patient Inference

### Command

```bash
python utils/inference_pipeline.py \
    --dicom-dir  <path/to/patient/dicom/series> \
    --config     config/config.yml \
    --model      model/best_dose_model_clinical_june23.pth \
    --dose-spacing 2.5
```

### Arguments

| Argument | Default | Description |
|---|---|---|
| `--dicom-dir` | — | Path to the folder containing CT + RTSTRUCT `.dcm` files |
| `--config` | `config/config.yml` | Path to the YAML configuration file |
| `--model` | `best_dose_model_clinical_june23.pth` | Model weights filename or path |
| `--dose-spacing` | CT slice spacing from config | Output RTDOSE grid spacing in mm (2.5 mm = ~65 MB, clinical standard) |
| `--keep-temp` | off | Keep the intermediate NIfTI workspace under `/tmp/` |

### Example (single patient)

```bash
python utils/inference_pipeline.py \
    --dicom-dir  testdata-23_06_2023/e9243f06.11a1.4a63.9b4d.cbef5c08a445/1.2.826.0.1.3680043... \
    --config     config/config.yml \
    --model      model/best_dose_model_clinical_june23.pth \
    --dose-spacing 2.5
```

### Expected Output (single patient)

```
  RTDOSE saved to: <dicom-dir>/predicted-dose-YYYYMMDD_HHMMSS.dcm
```

A `RP.dummy_plan.dcm` is also written into the same folder if no RTPLAN was present.

---

## 3. Batch Inference

Batch mode processes **all patient sub-folders** under a root directory automatically.

### Folder Structure Expected

```
<batch-dir>/
├── patient_A/
│   └── series_uid_A/
│       ├── CT_slice_001.dcm
│       ├── CT_slice_002.dcm
│       └── RTSTRUCT.dcm
├── patient_B/
│   └── series_uid_B/
│       ├── CT_slice_001.dcm
│       └── RTSTRUCT.dcm
└── ...
```

Each direct sub-folder of `--batch-dir` is treated as one patient. The pipeline descends
one level into any sub-folder to find the actual DICOM series directory.

### Command

```bash
python utils/inference_pipeline.py \
    --batch-dir  <path/to/patients/root> \
    --config     config/config.yml \
    --model      model/best_dose_model_clinical_june23.pth \
    --dose-spacing 2.5
```

### Example (batch — 5 patients)

```bash
python utils/inference_pipeline.py \
    --batch-dir  testdata-23_06_2023 \
    --config     config/config.yml \
    --model      model/best_dose_model_clinical_june23.pth \
    --dose-spacing 2.5
```

### Batch Summary Output

After all patients are processed, a summary table is printed:

```
============================================================
  BATCH SUMMARY
============================================================
  [OK    ] patient_A   <dicom-dir>/predicted-dose-20260625_110801.dcm
  [OK    ] patient_B   <dicom-dir>/predicted-dose-20260625_110923.dcm
  [FAILED] patient_C   ERROR: No RTSTRUCT found in ...
```

Failures are caught per-patient; the batch continues regardless.

---

## 4. Pipeline Stages (Internal)

Each call to `run_pipeline()` executes the following steps:

### [1] Workspace creation
A temporary directory is created under `/tmp/dose_infer_<random>/` to hold intermediate NIfTI channels during preprocessing and inference.

### [2] DICOM Preprocessing
`preprocess_dicom()` is called on the patient DICOM folder:
- CT volume is loaded and resampled to `target_spacing`
- RTSTRUCT contours are decoded into binary masks
- BEV beam mask is generated from RTPLAN (or 7 equispaced default beams)
- Signed distance maps, PTV discrete map, and all 7 input channels are written as NIfTIs

### [3] Neural Network Inference
`run_inference()` loads the model weights, runs the sliding-window 3-D U-Net forward pass,
applies body masking, and inverts the spatial transforms to restore the prediction to the
native CT grid via MONAI `Invertd`.

The predicted dose NIfTI is saved to the temp workspace.

### [4] RTDOSE DICOM Construction
`nifti_to_rtdose_dicom()` (from `utils/nifti_to_rtdose.py`) resamples the predicted dose
onto a coarser clinical dose grid (default 2.5 mm), encodes values as 16-bit integers with
a `DoseGridScaling` factor, and writes a fully-compliant RTDOSE DICOM that shares the
patient's StudyInstanceUID, FrameOfReferenceUID, and RTSTRUCT linkage.

**Automatic dummy RTPLAN creation:** Before writing the RTDOSE, the pipeline scans the
DICOM folder for an existing RTPLAN. If none is found, `create_dummy_plan_dicom()` (from
`utils/create_dummy_plan.py`) generates a minimal `RP.dummy_plan.dcm` using the PTV
centroid from the RTSTRUCT as the isocenter. This satisfies the mandatory
`ReferencedRTPlanSequence` tag required by most TPS systems.

### [5] Cleanup
The temporary workspace is deleted unless `--keep-temp` is specified.

---

## 5. Output Files

| File | Location | Description |
|---|---|---|
| `predicted-dose-<timestamp>.dcm` | Patient DICOM series folder | RTDOSE DICOM, importable into Eclipse/Monaco/RayStation |
| `RP.dummy_plan.dcm` | Patient DICOM series folder | Dummy RTPLAN (created only if no RTPLAN present) |
| `<case>_predicted_dose.nii.gz` | `/tmp/dose_infer_<id>/` (deleted unless `--keep-temp`) | Intermediate NIfTI at native CT grid |

---

## 6. DICOM Compatibility Notes

The generated RTDOSE is structured to load correctly in standard viewers:

| DICOM Tag | Value | Purpose |
|---|---|---|
| `StudyInstanceUID` | Copied from CT | Links dose to the same study |
| `FrameOfReferenceUID` | Copied from CT | Ensures spatial co-registration |
| `ReferencedStructureSetSequence` | Points to RTSTRUCT SOP UID | Links dose to contours |
| `ReferencedRTPlanSequence` | Points to RTPLAN SOP UID | Required for TPS import |
| `DoseGridScaling` | Gy / count | Converts stored uint16 → Gy |
| `DoseUnits` | `GY` | Physical dose in Gray |
| `DoseSummationType` | `PLAN` | Standard RTDOSE type |

---

## 7. Dummy RTPLAN Details

When no clinical RTPLAN exists, `create_dummy_plan_dicom()` creates `RP.dummy_plan.dcm` with:

- **Isocenter** — computed as the centroid of all PTV contour points from the RTSTRUCT
- **Beam geometry** — single 6 MV dynamic field (gantry 179°→181°, CC rotation)
- **Institution** — static values matching the clinical template (configurable in `utils/create_dummy_plan.py`)
- **Patient/Study/FrameOfReference UIDs** — copied from CT to ensure TPS linkage

> The dummy plan is marked `ApprovalStatus = UNAPPROVED` and is purely for structural
> DICOM linkage. It does not represent a real treatment plan.

---

## 8. Model Files

| Model file | Training date | Notes |
|---|---|---|
| `best_dose_model_physics.pth` | Ongoing | Latest checkpoint from active training run |
| `best_dose_model_clinical_june23.pth` | June 2023 | Clinically validated checkpoint for batch inference |

---

## 9. Common Issues

| Symptom | Cause | Fix |
|---|---|---|
| `ERROR: No RTSTRUCT found` | RTSTRUCT was deleted or is in an unexpected subfolder | Ensure RTSTRUCT `.dcm` is present anywhere under the patient folder tree |
| `nifti_to_rtdose_dicom not available. Skipped.` | Import error for `nifti_to_rtdose.py` | Ensure `utils/nifti_to_rtdose.py` exists; check that `_ROOT` is on `sys.path` |
| `Source image size [A] does not match [B]` (SimpleITK) | Axis order mismatch between MONAI output and SimpleITK | Fixed: Invertd output is transposed `(x,y,z)→(z,y,x)` before `GetImageFromArray` |
| `WARNING: RTPLAN not linked` in RTDOSE | No RTPLAN found and `create_dummy_plan` unavailable | Run `python utils/create_dummy_plan.py --dicom-dir <path>` manually first |
| Slow preprocessing for large batches | `_generate_bev_beam_mask` is CPU-bound for large volumes | Expected; runs once per patient during preprocessing |
