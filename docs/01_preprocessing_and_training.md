# Preprocessing & Training

## Overview

This document describes the full pipeline for converting raw clinical DICOM data into a
trained dose-prediction model. The pipeline has three major stages:

1. **DICOM → NIfTI preprocessing** — extracts multi-channel spatial inputs from CT + RTSTRUCT
2. **Dataset construction** — organises processed cases into a versioned folder structure
3. **Model training** — physics-guided neural network optimisation with online validation

---

## 1. Directory Layout

```
01 ICON/
├── config/
│   └── config.yml              # All hyperparameters and structural settings
├── preprocess/
│   └── Dataset001_ProstateDose/
│       ├── imagesTr/           # Multi-channel NIfTI inputs  (case_XXXX_CCCC.nii.gz)
│       ├── labelsTr/           # Ground-truth dose NIfTIs    (case_XXXX.nii.gz)
│       └── persistent_cache_physics/   # MONAI PersistentDataset cache
├── utils/
│   ├── training.py             # Main training entry-point
│   └── inference_pipeline.py  # Inference entry-point
└── model/
    └── best_dose_model_*.pth   # Saved model checkpoints
```

---

## 2. Configuration File — `config/config.yml`

All pipeline behaviour is driven by `config.yml`. Key sections:

| Section | Key fields | Purpose |
|---|---|---|
| `dataset` | `data_dir`, `target_spacing`, `patch_size` | Dataset root and resampling grid |
| `channels` | `index`, `role`, `modality` | Defines the 7 input channels and their MONAI transforms |
| `clinical_targets` | `prescription_dose_gy`, `ptv_structures` | PTV labels and prescription dose levels |
| `training` | `epochs`, `batch_size`, `learning_rate` | Optimiser settings |
| `target_lambdas` | `mse`, `mandatory`, `ptv`, `ring`, … | Per-term loss weights |
| `oars` | list of OAR name patterns, dose limits | Structures-at-risk constraints |

> **Important:** Changing `rx_gy` or any channel definition invalidates the on-disk cache.
> Delete `persistent_cache_physics/` after any config change that affects transform outputs.

---

## 3. Input DICOM Requirements

Each patient case must supply the following DICOM series inside a single folder tree:

| Modality | Required | Notes |
|---|---|---|
| **CT** | ✅ Yes | Axial slices, any slice thickness (resampled internally) |
| **RTSTRUCT** | ✅ Yes | Must contain PTV + OAR contours (see structure names below) |
| **RTPLAN** | Optional | Used for beam geometry; 7-field equispaced default if absent |
| **RTDOSE** | Optional | Used as ground-truth label during training |

### Expected Structure Names

The pipeline uses fuzzy matching (case-insensitive, partial match) for the following roles:

| Role | Example names matched |
|---|---|
| PTV high-dose | `PTV_62/20`, `PTV62`, `CTV_62`, `PTV_62/20 PLAN` |
| PTV low-dose | `PTV_44/20`, `PTV44`, `CTV_44` |
| Body | `BODY`, `External` |
| Bladder | `Bladder` |
| Anorectum | `Anorectum`, `Rectum` |
| Penile Bulb | `PenileBulb`, `Penile_Bulb` |
| Bowel Bag | `Bag_Bowel`, `BowelBag` |
| Femur (bilateral) | `Femur_Head_L`, `Femur_Head_R` |

---

## 4. Input Channels

The model receives **7 spatial channels** per case, each as a 3-D NIfTI at the target grid:

| Channel index | Role | Description |
|---|---|---|
| `0000` | CT | Normalised Hounsfield Units |
| `0001` | PTV discrete | Voxel-wise prescription dose encoding (`rx_gy` per PTV level) |
| `0002` | Body binary | 1 inside body contour, 0 outside |
| `0003` | BEV beam mask | Beam's-eye-view projection of treatment beams |
| `0004` | SDM — PTV | Signed distance map for PTV surface |
| `0005` | SDM — OARs | Signed distance map for combined OAR surfaces |
| `0006` | BEV depth | Cumulative radiological depth along beam paths |

---

## 5. Preprocessing

Preprocessing is performed automatically when training calls `preprocess_dicom()` inside
`utils/inference_pipeline.py`. To manually preprocess a single patient for dataset
construction, use the same function from Python:

```python
from utils.inference_pipeline import preprocess_dicom
import yaml

with open("config/config.yml") as f:
    config = yaml.safe_load(f)

preprocess_dicom(
    dicom_dir  = "path/to/patient/dicom/series",
    images_dir = "preprocess/Dataset001_ProstateDose/imagesTr",
    config     = config,
    case_name  = "case_0001",
)
```

### Preprocessing Steps (internal)

1. **CT load** — reads all CT slices via `pydicom`, stacks into a 3-D numpy volume.
2. **RTSTRUCT decode** — extracts binary masks for each named structure onto the CT grid.
3. **Spacing resample** — resamples all volumes to `target_spacing` (default `1.27 × 1.27 × 2.5 mm`).
4. **BEV beam mask** — projects gantry beams from RTPLAN (or 7 equispaced default angles) to produce a 3-D fluence aperture mask.
5. **Signed distance maps** — `scipy.ndimage.distance_transform_edt` computes interior/exterior distances for PTV and OAR contours.
6. **PTV discrete map** — encodes each PTV voxel with its `rx_gy` prescription value.
7. **Channel save** — writes each channel as `<case_name>_<CCCC>.nii.gz` to `imagesTr/`.

---

## 6. Training

### Launch

Run from the `01 ICON/` directory:

```bash
python utils/training.py
```

No additional command-line arguments are required; all settings are read from `config/config.yml`.

### What happens during training

1. **Dataset scan** — discovers all `case_XXXX_0000.nii.gz` files under `imagesTr/` and
   builds a patient list. Cases with fewer than `min_channels` files are skipped.
2. **MONAI PersistentDataset** — applies the full Compose transform chain on first access
   and caches the result to `persistent_cache_physics/`. Subsequent epochs read from cache.
3. **Train / validation split** — last N cases (configurable) are held out for validation.
4. **Forward pass** — 3-D U-Net predicts a single-channel dose volume.
5. **Physics-guided loss** — weighted sum of:
   - `mse` — voxel-wise MSE vs ground-truth dose
   - `mandatory` — penalises predicted dose inside PTV below prescription
   - `ptv` — soft constraint on PTV D95
   - `ptv_max` — ceiling constraint on maximum PTV dose
   - `ring` — penalises dose outside body beyond a margin ring
   - `anticollapse` — prevents trivially zero predictions
   - `homogeneity` — promotes uniform dose within PTV
   - `laplacian` / `smooth` — spatial regularity terms
6. **Validation metrics** (logged to stdout each epoch):

   | Metric | Description |
   |---|---|
   | `PTV_62_D95` | Dose received by 95 % of high-dose PTV (Gy) |
   | `PTV_62_Mean` | Mean dose inside high-dose PTV (Gy) |
   | `PTV_62_HI` | Homogeneity Index `(D2 − D98) / D50` |
   | `PTV_62_CI` | Conformity Index (fraction of prescription isodose inside PTV) |
   | `PTV_44_D95` | D95 for low-dose PTV (Gy) |
   | `Bladder_Mean`, `Anorectum_Mean`, etc. | Mean OAR doses (Gy) |

7. **Checkpoint** — best model (lowest combined validation loss) saved to
   `model/best_dose_model_physics.pth`.

### Cache Invalidation

The training script computes a SHA-256 hash of `config.yml` and stores it alongside the
cache. If the hash changes (i.e., config was edited), the old cache is automatically
invalidated and rebuilt from scratch.

To force a manual cache clear:

```bash
rm -rf preprocess/Dataset001_ProstateDose/persistent_cache_physics/
```

---

## 7. Common Issues

| Symptom | Cause | Fix |
|---|---|---|
| `Found 0 patients` | Wrong `data_dir` in config, or `imagesTr/` is empty | Check `data_dir` points to the correct dataset root |
| `PTV D95 = 0 Gy` in validation | Stale cache with wrong `rx_gy` encoding | Clear cache and retrain |
| `PytorchStreamReader failed reading zip archive` | Corrupted cache file (interrupted training) | Clear cache directory |
| `NameError: eval_device` | Variable defined inside GPU branch but used outside | Ensure `eval_device` is set unconditionally before the validation loop |
