# TBIMH-ProtoFuse

**Foundation-Enhanced Gaussian Prototypical Multimodal Fusion for Baseline Prognostic Prediction of Type B Aortic Intramural Hematoma**

This repository is the planned official implementation of **TBIMH-ProtoFuse**, a baseline-only multimodal prognostic framework for predicting whether a patient with type B aortic intramural hematoma (TBIMH) will show **hematoma progression** or **absorption/non-progression** within 15 months after the baseline computed tomography angiography (CTA) examination.

> **Release status.** The manuscript is currently under peer review. To comply with institutional review, ethics, and journal policies, the full source code will be released after the paper is formally published. The current pre-publication repository is not yet a runnable code release; this README documents the exact reproduction interface that will be provided, including the expected directory layout, preprocessing scripts, training commands, evaluation protocol, and output files.

![TBIMH-ProtoFuse framework](Figure/3.png)

## What This Project Does

TBIMH-ProtoFuse estimates 15-month TBIMH progression risk using only information available at the initial clinical decision point:

- structured baseline clinical variables;
- baseline thoracoabdominal CTA;
- CTA-derived radiomic features from the baseline aortic region of interest;
- baseline CTA report text, when available.

Follow-up CTA scans, post-baseline laboratory tests, treatment updates, later clinical notes, and progression reports are **not** used as model inputs. They are used only to adjudicate the 15-month endpoint.

The model outputs:

- `P(progression within 15 months)`;
- `P(absorption/non-progression within 15 months)`;
- binary risk prediction under a specified threshold;
- patient-level modality weights for clinical, CTA, and text evidence.

## Clinical Prediction Target

The event class is **15-month hematoma progression**. Progression is defined as new or aggravated aortic deterioration on follow-up CTA, including one or more of:

- new ulcer-like projection;
- maximal aortic diameter increase of at least 5 mm;
- classical or localized dissection;
- impending rupture;
- aneurysm formation.

Absorption/non-progression is defined as partial/complete hematoma absorption or stable hematoma morphology without any progression feature within the 15-month window.

## Cohort Used in the Paper

The study cohort contains 309 retrospectively collected TBIMH patients from three hospitals in Jiangxi, China:

| Center | Patients |
| --- | ---: |
| Hospital A: The Second Affiliated Hospital of Nanchang University | 240 |
| Hospital B: Jiangxi Provincial People's Hospital | 12 |
| Hospital C: Ganzhou People's Hospital | 57 |

Outcome distribution:

| Outcome | Patients |
| --- | ---: |
| Progression within 15 months | 183 |
| Absorption/non-progression within 15 months | 126 |

Raw CTA images and clinical records contain protected health information and cannot be openly redistributed. After publication, the repository will release all source code and, where permitted by ethics approval and institutional data-use constraints, de-identified derived features, split files, prediction outputs, and example/demo data.

## Model Overview

TBIMH-ProtoFuse follows a five-stage workflow.

1. **Baseline evidence construction**

   Clinical variables are imputed, standardized, one-hot encoded, and augmented with missingness indicators. Baseline CTA ROIs are used for radiomic extraction and ROI-level patch sampling. Baseline CTA report text is cleaned to retain only the initial Findings and Impression/Conclusion sections.

2. **Modality-specific encoding**

   The clinical branch uses a tabular foundation prior. The CTA branch combines harmonized radiomics and frozen self-supervised CTA visual embeddings. The optional text branch encodes baseline CTA report text and is masked when no report text is available.

3. **Uncertainty-aware gated fusion**

   Each modality produces a branch-specific prediction and uncertainty estimate. A masked gating module assigns patient-specific weights to available modalities.

4. **Gaussian prototypical prediction**

   Progression and absorption/non-progression are represented by class-specific Gaussian prototypes in the fused prognostic embedding space.

5. **Optimization and calibration**

   The model is trained with classification, prototype, supervised contrastive, modality-consistency, center-adversarial, and calibration losses, followed by temperature calibration inside the training/development folds.

Default settings reported in the paper include:

| Component | Setting |
| --- | --- |
| Shared embedding dimension | `128` |
| CTA ROI patches per patient | `16` |
| Patch size | `224 x 224` |
| Support samples per class | `8` |
| Query samples per class | `8` |
| Prototype covariance regularizer | `1e-4` |
| Optimizer | `AdamW` |
| Initial learning rate | `1e-4` |
| Weight decay | `1e-5` |
| Maximum epochs | `200` |

## Planned Repository Structure

The public code release will use the following structure.

```text
TBIMH-ProtoFuse/
  README.md
  LICENSE
  requirements.txt
  configs/
    tbimh_protofuse.yaml
    cv_10x5fold.yaml
    external_hospital_a_to_bc.yaml
    low_data_20pct.yaml
    low_data_40pct.yaml
    ablation/
  data/
    README.md
    demo/
    splits/
      repeated_5fold_10seeds.json
      hospital_a_to_bc.json
  src/
    data/
      clinical_preprocess.py
      radiomics_preprocess.py
      report_preprocess.py
      patch_sampler.py
      split_builder.py
    models/
      clinical_encoder.py
      cta_encoder.py
      text_encoder.py
      uncertainty_gate.py
      gaussian_prototype.py
      protofuse.py
    training/
      train_cv.py
      train_external.py
      losses.py
      calibration.py
    evaluation/
      metrics.py
      calibration_curve.py
      decision_curve.py
      statistical_tests.py
    utils/
  scripts/
    00_prepare_splits.py
    01_prepare_clinical.py
    02_extract_radiomics.py
    03_sample_cta_patches.py
    04_cache_cta_embeddings.py
    05_train_protofuse_cv.py
    06_eval_protofuse_cv.py
    07_train_external_center.py
    08_run_low_data.py
    09_run_ablation.py
    10_run_roi_perturbation.py
    11_infer_single_patient.py
  outputs/
    cv/
    external/
    low_data/
    ablation/
    robustness/
```

## Expected Data Layout

For full reproduction with local authorized data, the code will expect the following layout.

```text
data/private/
  clinical.csv
  reports.csv
  labels.csv
  centers.csv
  dicom/
    <patient_id>/
      *.dcm
  masks/
    <patient_id>.nii.gz

data/derived/
  clinical_processed/
  radiomics_raw.csv
  radiomics_harmonized/
  cta_patches/
  cta_embeddings/
  text_embeddings/
```

Required patient-level identifiers:

| File | Required columns |
| --- | --- |
| `clinical.csv` | `patient_id`, baseline demographics, symptoms, comorbidities, medication history, hemodynamic indicators, admission laboratory tests |
| `reports.csv` | `patient_id`, cleaned baseline CTA Findings and Impression/Conclusion text |
| `labels.csv` | `patient_id`, `label`, where `0 = progression` and `1 = absorption/non-progression` |
| `centers.csv` | `patient_id`, `center`, where centers are encoded as Hospital A/B/C or equivalent site IDs |

Follow-up information should appear only in the endpoint-adjudication file used to create `labels.csv`. It should not be included in any model input file.

## Environment

The paper experiments used:

- Python `3.10`;
- PyTorch `2.1`;
- CUDA `12.1`;
- Ubuntu `22.04`;
- Intel Xeon Gold 6330 CPU;
- 256 GB system memory;
- 4 x NVIDIA A800 GPUs with 80 GB memory.

The released code will also support single-GPU execution for most experiments. CTA feature extraction and repeated cross-validation are the most computationally expensive steps.

Planned installation:

```bash
conda create -n tbimh-protofuse python=3.10 -y
conda activate tbimh-protofuse
pip install -r requirements.txt
```

## Reproduction Workflow

The following commands describe the public interface that will be released with the source code.

### 1. Build Repeated Five-Fold Splits

```bash
python scripts/00_prepare_splits.py \
  --labels data/private/labels.csv \
  --centers data/private/centers.csv \
  --n_repeats 10 \
  --n_folds 5 \
  --out data/splits/repeated_5fold_10seeds.json
```

### 2. Prepare Baseline Clinical Variables

All imputation, scaling, one-hot encoding, missingness indicators, and feature statistics are fitted inside each training/development fold.

```bash
python scripts/01_prepare_clinical.py \
  --clinical_csv data/private/clinical.csv \
  --labels_csv data/private/labels.csv \
  --splits data/splits/repeated_5fold_10seeds.json \
  --out_dir data/derived/clinical_processed
```

### 3. Extract CTA Radiomics From Baseline ROI

The manuscript used 3D Slicer `5.2.2` and the Slicer Radiomics plugin version `aa418a5` for ROI-based radiomic extraction. The code release will include wrappers and configuration files for reproducible extraction.

```bash
python scripts/02_extract_radiomics.py \
  --dicom_dir data/private/dicom \
  --mask_dir data/private/masks \
  --out_csv data/derived/radiomics_raw.csv \
  --slicer_version 5.2.2
```

### 4. Sample ROI-Level CTA Patches

By default, 16 representative 2D ROI patches are sampled from each baseline CTA and resized to `224 x 224`.

```bash
python scripts/03_sample_cta_patches.py \
  --dicom_dir data/private/dicom \
  --mask_dir data/private/masks \
  --patches_per_patient 16 \
  --patch_size 224 \
  --out_dir data/derived/cta_patches
```

### 5. Cache Frozen CTA Embeddings

The self-supervised CTA visual backbone is frozen. Caching patch embeddings is recommended for repeated cross-validation.

```bash
python scripts/04_cache_cta_embeddings.py \
  --patch_dir data/derived/cta_patches \
  --config configs/tbimh_protofuse.yaml \
  --out_dir data/derived/cta_embeddings
```

### 6. Train TBIMH-ProtoFuse With Repeated 5-Fold CV

```bash
python scripts/05_train_protofuse_cv.py \
  --config configs/tbimh_protofuse.yaml \
  --splits data/splits/repeated_5fold_10seeds.json \
  --clinical_dir data/derived/clinical_processed \
  --radiomics_csv data/derived/radiomics_raw.csv \
  --cta_embedding_dir data/derived/cta_embeddings \
  --reports_csv data/private/reports.csv \
  --labels_csv data/private/labels.csv \
  --centers_csv data/private/centers.csv \
  --out_dir outputs/cv/tbimh_protofuse
```

### 7. Evaluate Cross-Validation Results

```bash
python scripts/06_eval_protofuse_cv.py \
  --run_dir outputs/cv/tbimh_protofuse \
  --labels_csv data/private/labels.csv \
  --out_dir outputs/cv/tbimh_protofuse/evaluation
```

Expected output files:

```text
outputs/cv/tbimh_protofuse/
  fold_checkpoints/
  predictions/
    oof_predictions.csv
    modality_weights.csv
  evaluation/
    metrics_summary.csv
    calibration_bins.csv
    decision_curve.csv
    performance_table.md
```

### 8. External-Center Validation

The paper trains on Hospital A and evaluates directly on Hospital B + Hospital C. No Hospital B/C preprocessing, harmonization, calibration, threshold, or label information is used during model development.

```bash
python scripts/07_train_external_center.py \
  --config configs/external_hospital_a_to_bc.yaml \
  --train_centers A \
  --test_centers B C \
  --clinical_csv data/private/clinical.csv \
  --radiomics_csv data/derived/radiomics_raw.csv \
  --cta_embedding_dir data/derived/cta_embeddings \
  --reports_csv data/private/reports.csv \
  --labels_csv data/private/labels.csv \
  --centers_csv data/private/centers.csv \
  --out_dir outputs/external/hospital_a_to_bc
```

### 9. Low-Data Stress Tests

```bash
python scripts/08_run_low_data.py \
  --config configs/low_data_20pct.yaml \
  --train_fraction 0.20 \
  --splits data/splits/repeated_5fold_10seeds.json \
  --out_dir outputs/low_data/20pct

python scripts/08_run_low_data.py \
  --config configs/low_data_40pct.yaml \
  --train_fraction 0.40 \
  --splits data/splits/repeated_5fold_10seeds.json \
  --out_dir outputs/low_data/40pct
```

### 10. Ablation and ROI Perturbation

```bash
python scripts/09_run_ablation.py \
  --config configs/tbimh_protofuse.yaml \
  --splits data/splits/repeated_5fold_10seeds.json \
  --ablation_config configs/ablation/all_components.yaml \
  --out_dir outputs/ablation

python scripts/10_run_roi_perturbation.py \
  --config configs/tbimh_protofuse.yaml \
  --sigmas 0.0 0.5 1.0 1.5 2.0 \
  --dicom_dir data/private/dicom \
  --mask_dir data/private/masks \
  --out_dir outputs/robustness/roi_perturbation
```

### 11. Single-Patient Inference

```bash
python scripts/11_infer_single_patient.py \
  --checkpoint outputs/cv/tbimh_protofuse/fold_checkpoints/best.pt \
  --clinical_json examples/single_patient/clinical.json \
  --cta_dicom_dir examples/single_patient/dicom \
  --roi_mask examples/single_patient/mask.nii.gz \
  --report_txt examples/single_patient/baseline_report.txt \
  --out_json examples/single_patient/prediction.json
```

The output JSON will contain:

```json
{
  "patient_id": "example_001",
  "prob_progression_15m": 0.72,
  "prob_absorption_nonprogression_15m": 0.28,
  "predicted_class": "progression",
  "threshold": 0.5,
  "modality_weights": {
    "clinical": 0.31,
    "cta": 0.57,
    "text": 0.12
  }
}
```

## Leakage-Prevention Policy

The released code will enforce the same leakage-prevention protocol used in the manuscript.

- Continuous clinical variables are imputed and standardized using training-fold statistics only.
- Categorical encoders are fitted only on training/development folds.
- Radiomic near-zero-variance filtering and correlation filtering are fitted only within training/development folds.
- Radiomic harmonization parameters are estimated only from training/development data.
- Prototype construction uses only support samples from the current training partition.
- Threshold selection and temperature calibration use only training/development folds.
- Held-out folds and external-center cohorts are used only for final prediction and metric calculation.
- Follow-up CTA and post-baseline records are never provided to the model.

## Reported Main Results

In ten repeated stratified five-fold cross-validation, TBIMH-ProtoFuse achieved:

| Metric | Result |
| --- | ---: |
| Precision | 86.3% |
| Recall / sensitivity | 81.2% |
| Specificity | 79.7% |
| Accuracy | 80.5% |
| ROC-AUC | 88.9% |
| PR-AUC | 89.7% |
| Brier score | 14.8% |
| Expected calibration error | 5.4% |

In external-center validation, training on Hospital A and testing on Hospital B + Hospital C, TBIMH-ProtoFuse achieved:

| Metric | Result |
| --- | ---: |
| Precision | 84.6% |
| Recall / sensitivity | 89.7% |
| Specificity | 81.5% |
| Accuracy | 86.1% |
| ROC-AUC | 92.7% |
| PR-AUC | 93.4% |
| Brier score | 12.9% |
| Expected calibration error | 5.0% |

## Computational Cost

In the full multimodal configuration:

| Item | Value |
| --- | ---: |
| Total parameters | 122.1M |
| Trainable parameters | 1.94M |
| Trainable ratio | 1.59% |
| Peak GPU memory | 7.9 GB |
| Training time per fold | 10.8 min |
| Inference time per patient | 82.6 ms |
| Inference time with cached CTA embeddings | 8.9 ms |

Radiomic feature processing and harmonization required approximately 38 minutes for the full cohort, and self-supervised CTA feature precomputation required approximately 6 minutes. Cached CTA embeddings occupied less than 100 MB.

## Data and Code Availability

- **Source code:** to be released in this repository after the paper is formally published.
- **Raw clinical data and CTA scans:** not publicly redistributable because they contain sensitive patient information and are governed by institutional ethics approvals.
- **Derived features and model outputs:** de-identified radiomics, split files, prediction outputs, and demo data will be shared where permitted by ethics and institutional data-use constraints.
- **Pretrained/frozen encoders:** download instructions and checksums will be provided in the release notes when the code is made public.

## Citation

If you use this repository after release, please cite:

```bibtex
@article{zhao2026tbimhprotofuse,
  title   = {Foundation-Enhanced Gaussian Prototypical Multimodal Fusion for Baseline Prognostic Prediction of Type B Aortic Intramural Hematoma},
  author  = {Zhao, Wenpeng and Zhong, Baoliang and Zeng, Xiong and Hu, Yiliang and Huang, Jingzheng and Zhou, Weimin},
  journal = {IEEE Journal of Biomedical and Health Informatics},
  year    = {2026},
  note    = {Under review}
}
```

## Contact

For questions about the paper or future code release, please contact:

- Wenpeng Zhao: `wenpengzhao0412@163.com`
- Weimin Zhou: `ndefy09073@ncu.edu.cn`

## Disclaimer

TBIMH-ProtoFuse is a research prototype for retrospective prognostic modeling. It is not a medical device and should not be used for clinical decision-making without prospective validation, regulatory review, and institution-specific workflow evaluation.
