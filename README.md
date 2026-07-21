# CaFeR
Training-free frequency reparameterization of SMPL shape responses with six-region soft supports and a 610D linear decoder.

# CaFeR: Coupling-Aware Frequency Reparameterization for SMPL

Anonymous code release for paper review.

CaFeR is a training-free, explicit linear reparameterization of the SMPL shape space. It starts from the vertex-response fields induced by individual SMPL shape parameters, constructs a skeleton-calibrated six-region soft support, and combines global responses, regional response residuals, and localized spectral compensation into a 610-dimensional decoder. The method preserves the SMPL template topology, vertex correspondence, joint system, skinning weights, and linear blend skinning (LBS) interface.

The Linear-610 decoder contains

```text
10  global low-frequency response atoms
60  regional response atoms       = 6 regions x 10 atoms
540 regional spectral vectors     = 6 regions x 30 modes x 3 axes
----------------------------------------------------------------
610 coefficients
```

This repository also contains diagnostic experiments for response nonlocality, six-region support construction, dimension-matched representation capacity, LBS error propagation, sparse coefficient prediction, and external registered-geometry coverage.

## Important scope

- CaFeR assumes the standard SMPL topology with 6,890 vertices and known vertex correspondence.
- The method is a representation and fitting framework; it is not an image-to-mesh system, an unregistered point-cloud method, or a raw-scan registration pipeline.
- The deployable representation is the 610D linear decoder. `Oracle-Transition` and the Step3 target-derived high residual are diagnostic upper bounds and are not produced by the 610 coefficients alone.
- The main synthetic capacity comparison and the external DFAUST equal-capacity comparison use an unweighted fitting term. The separate final CaFeR experiment uses frozen nonuniform fitting weights and must not be mixed into the equal-capacity table.
- In this project, “boundary residual jump” means the change of the reconstruction residual field across Step2 region-boundary edges. It does not mean a mesh crack or topological discontinuity.

## Repository layout

```text
RA_SMPL/
├── step1/                         # SMPL single-parameter response diagnosis
├── step2/                         # six-region soft support and MRF refinement
├── step3/                         # current CaFeR construction and experiments
├── step4/                         # shared-LBS pose compatibility
├── step5/                         # sparse registered observations -> coefficients
├── step1_multi/                   # legacy exploratory scripts; not canonical
├── step3-1/                       # legacy Step3 snapshot; not canonical
├── check_dfaust_suitability.py
├── export_dfaust_balanced_large_scale.py
├── run_cafer_external_registered_targets.py
├── run_dfaust_equal_dim_610.py
└── visualization scripts
```

Use the extracted, outer `step1/` through `step5/` source files. Do not run the nested ZIP archives: some are redundant, while the nested Step1, Step2, and Step3 archives contain older files. In particular, `step3/` is the current 50-seed/1,000-target implementation; `step3-1/` is an older 5-seed snapshot.

### Main modules

| Module | Purpose | Canonical status |
|---|---|---|
| `step1` | Diagnose cross-region diffusion of the ten SMPL shape responses | Main |
| `step2` | Build skeleton-calibrated six-region soft supports and refine boundaries | Main |
| `step3` | Build Linear-610, run controlled targets, ablations, and fair baselines | Main |
| `step4` | Apply identical SMPL LBS transforms to targets and reconstructions | Main |
| `step5` | Predict normalized Linear-610 coefficients from registered sparse vertices | Main Ridge experiment |
| `step1_multi` | Early multi-beta visualization experiments | Legacy; excluded from paper reproduction |
| `step3-1` | Older Step3 target generator and output convention | Legacy; excluded from paper reproduction |

## Environment

The reference runs were performed on Windows with Python 3.8.20. The tested environment contained:

| Package | Tested version |
|---|---:|
| NumPy | 1.22.4 |
| SciPy | 1.10.1 |
| pandas | 2.0.3 |
| h5py | 3.11.0 |
| Matplotlib | 3.7.5 |
| Pillow | 10.4.0 |
| PyTorch | 1.9.0 CPU |
| scikit-learn | 1.3.2 |
| trimesh | 4.10.0 |
| Open3D | 0.19.0 |
| PyVista | 0.44.2 |
| smplx | installed from the official package |

CUDA is optional. Step1 and Step4 can fall back to CPU. PyVista/VTK and Open3D are needed mainly for mesh validation and visualization.

This source snapshot does not contain a pinned `requirements.txt` or Conda lock file. A minimal environment can be prepared as follows, with PyTorch installed separately using a build compatible with Python 3.8:

```powershell
conda create -n cafer python=3.8
conda activate cafer
pip install numpy scipy pandas h5py matplotlib pillow scikit-learn trimesh pyvista open3d smplx
```

## Licensed assets and expected data layout

SMPL model files, DFAUST registrations, and AMASS data are not distributed with this code. Download them from their official sources and comply with their respective licenses.

The current scripts expect a data root with the following logical layout:

```text
<DATA_ROOT>/
├── smpl_project/
│   ├── RA_SMPL/                   # this repository
│   └── models/
│       └── smpl/
│           ├── SMPL_NEUTRAL.pkl
│           ├── SMPL_FEMALE.pkl    # external gendered baseline only
│           └── SMPL_MALE.pkl      # external gendered baseline only
├── dataset/
│   └── DFAUST/
│       ├── registrations_f.hdf5
│       └── registrations_m.hdf5
└── result/                         # generated caches and experiment outputs
```

The neutral model must use the standard SMPL vertex order, 24-joint convention, and ten shape coefficients. Do not substitute a different topology without rebuilding and revalidating every support, basis, and case.

### Path configuration

Most current Step2, Step3, Step4, and equal-capacity scripts support environment-based or command-line path overrides. In PowerShell:

```powershell
$env:DIGITAL_MAN_BASE_DIR = "<DATA_ROOT>"
```

Additional Step4 overrides are:

```powershell
$env:CAFER_STEP3_CASE_ROOT = "<STEP3_CASES>"
$env:CAFER_STEP2_SUPPORT_ROOT = "<STEP2_SUPPORT>"
$env:CAFER_STEP4_OUTPUT_ROOT = "<STEP4_OUTPUT>"
$env:CAFER_DEVICE = "cpu"
```

Step1 still defines `BASE_DIR` directly in `step1/step1_config.py`; update that one value before a clean Step1 run. The DFAUST exporter and weighted external runner accept path arguments. Step5 should be run with explicit output directories, as shown below, because its default output root is currently local-machine specific.

## Reproduction pipeline

Run commands from the repository root unless noted otherwise.

### 1. Diagnose SMPL response diffusion

Step1 uses the neutral SMPL model at zero pose and perturbs each of the ten shape coefficients at nine values from -3.00 to +3.00. The four coarse diagnostic regions are torso, upper limbs, lower limbs, and head.

```powershell
python step1/step1_region_partition.py
python step1/step1_perturbation.py
python step1/step1_coupling_analysis.py
python step1/step1_distribution_validation.py
python step1/step1_visualization.py
```

Optional mesh exports:

```powershell
python step1/step1_3d_comparison.py
python step1/step1_render_beta_response_meshes.py
```

The main diagnostics are normalized response entropy, the 95% response-support ratio, and inter-region directional coupling. The quantity used here is the squared deformation response, `||Delta v||^2`; it is distinct from the Step2 MRF energy.

Important outputs include:

```text
<RESULT_ROOT>/vertex_region_lbs.npy
<RESULT_ROOT>/vertex_region_lbs_soft.npy
<RESULT_ROOT>/boundary_mask_lbs.npy
<RESULT_ROOT>/perturb_results/
<RESULT_ROOT>/coupling_metrics.csv
<RESULT_ROOT>/coupling_region_pairs.csv
<RESULT_ROOT>/step1_analysis/
```

The distribution validation draws 500 synthetic beta samples. It is not a real-scan experiment.

### 2. Construct six-region soft supports

Step2 splits the upper and lower limbs into proximal and distal portions using skeleton-chain coordinates, calibrates their soft mass to skeleton-length ratios, and performs lightweight structure-aware MRF refinement.

```powershell
python step2/deformation_field.py
python step2/gs_gcp_final.py
python step2/step2_validate_results.py
python step2/step2_visualization.py
python step2/step2_robustness.py
```

`gs_gcp_final.py` can rebuild the deformation-sensitivity field when the cache is missing, but running `deformation_field.py` explicitly makes provenance easier to audit. For the formal pipeline, confirm that the Step1 file `vertex_region_lbs_soft.npy` was loaded rather than relying on the fallback reconstruction.

The six regions are:

```text
torso
upper proximal limbs
upper distal limbs
lower proximal limbs
lower distal limbs
head
```

Core assets are written to:

```text
<RESULT_ROOT>/GS_GCP_SRMP_FINAL_6REGION/
├── soft_initial.npy
├── soft_mrf.npy
├── soft_final.npy
├── label_final.npy
├── hard_core_mask.npy
├── transition_mask.npy
├── boundary_edges.npy
├── deformation_field_Dv.npy
├── pca_train_weights.npy
└── tables/
```

The robustness script injects synthetic label noise and uses a simplified Potts diagnostic. It does not use human-annotated regional ground truth and is not the complete Step2 MRF.

### 3. Build and evaluate Linear-610

Step3 loads the ten unit response atoms and the frozen Step2 support. It builds the normalized basis once per `ModelSpec`, creates a solver cache, and reuses its Cholesky factorization when numerically valid.

Recommended current Step3 sequence:

```powershell
python step3/run_single.py
python step3/audit_step3_target_validity_relative.py
python step3/run_final.py
python step3/run_ablation_subset.py
python step3/run_boundary_sweep.py
python step3/run_solver_check.py
python step3/run_tables.py
```

`python step3/run_all.py` runs the single case, final evaluation, subset ablation, boundary sweep, and tables, but it does not run the target-validity audit or solver check.

The current default uses 50 seeds and produces 1,000 controlled, topology-consistent targets:

```text
50  global SMPL
50  transition
300 localized response
300 spectral local
300 mixed
```

The final weighted Step3 specification is fixed as:

```text
topk regional atoms       = 10 per region
spectral modes            = 30 per region
leakage regularization    = off
boundary regularization   = off
deformation weighting     = on
body-core weighting       = on
transition weighting      = on
```

All basis vectors are L2-normalized after flattening. Use `coeff.npy` with the normalized basis. `physical_coeff.npy` is a raw-basis-scale diagnostic and must not be decoded with the normalized basis.

Frozen Step3 Ridge values are:

```text
Direct beta                       1e-6
CaFeR global response block       1e-6
CaFeR regional response atoms     1e-5
CaFeR regional spectral vectors   1e-4
Global spectral baseline          1e-5  (generic-group fallback)
Hard/Soft local spectral baseline 1e-4
```

The exported decoder is stored under:

```text
<RESULT_ROOT>/step3_cafer_sr_validated_targets/final_k30_noleak_noboundary/linear610_decoder/
```

### 4. Run the dimension-matched synthetic comparison

The final required fair-capacity run is:

```powershell
python step3/run_fair_baselines_all_required.py
```

For a smaller smoke test:

```powershell
python step3/run_fair_baselines_debug.py
```

The fair protocol fixes:

```text
data weights                  W = I
deformation-sensitivity weight off
body-core weight               off
transition weight              off
leakage regularization         0
boundary regularization        0
target-derived high residual   off
```

It evaluates Direct beta and Global, Hard-mask local, Soft-mask local, and CaFeR representations at the configured dimensions. The formal 610D comparison uses the same 1,000 targets, solver implementation, metrics, and column-normalization rule for all four 610D representations.

Main outputs:

```text
<RESULT_ROOT>/step3_cafer_sr_validated_targets/
└── fair_dimension_baselines_all_required/synthetic_capacity/
    ├── models.csv
    ├── target_grid.csv
    ├── run_manifest.json
    ├── all_metrics.csv
    ├── synthetic_summary_by_model.csv
    ├── synthetic_summary_by_model_and_target.csv
    ├── synthetic_paper_table_core.csv
    ├── synthetic_cafer_allocation_ablation.csv
    └── run_complete.json
```

### 5. Evaluate shared-LBS pose compatibility

Use evaluation mode for the quantitative experiment:

```powershell
python step4/run_step4.py --mode eval
```

Optional visualization modes are:

```powershell
python step4/run_step4.py --mode visuals
python step4/run_step4.py --mode all
```

The default `all` mode can export tens of thousands of OBJ/PLY files for the full 1,000-case run. Use it only when those visuals are required.

Step4 applies seven fixed diagnostic poses to each target and reconstruction using the same SMPL `posedirs`, joint regressor, kinematic tree, and skinning weights. These are manually defined diagnostic poses, not motion-capture sequences. The pure Linear-610 result is the `before_high` reconstruction; the `final` reconstruction includes the target-derived correction and should be reported separately.

Main tables:

```text
<STEP4_OUTPUT>/tables/step4_pose_consistency_all.csv
<STEP4_OUTPUT>/tables/step4_summary_by_pose.csv
<STEP4_OUTPUT>/tables/step4_summary_by_target_type.csv
```

### 6. Evaluate sparse coefficient prediction

The formal Step5 experiment predicts normalized Linear-610 coefficients from registered sparse canonical vertex displacements using standardized Ridge regression. The MLP implementation is optional and is not used for the reported Ridge result.

For the 512-vertex paper setting, use distinct output directories:

```powershell
python step5/run_step5.py `
  --stage all `
  --sparse_count 512 `
  --step3_cases_root "<STEP3_CASES>" `
  --step2_root "<STEP2_SUPPORT>" `
  --decoder_dir "<STEP5_OUTPUT>/linear610_decoder" `
  --sparse_dir "<STEP5_OUTPUT>/sparse" `
  --dataset_dir "<STEP5_OUTPUT>/dataset_linear610" `
  --model_dir "<STEP5_OUTPUT>/models" `
  --eval_dir "<STEP5_OUTPUT>/eval" `
  --force
```

Do not add `--allow_empirical_decoder` for a formal run. That option is a debug-only fallback and does not guarantee the exact Step3 basis. Do not add `--run_mlp` when reproducing the reported Ridge result.

Formal checks should confirm:

```text
explicit Step3 decoder found
coefficient dimension = 610
decoder-to-solver MPVPE <= 0.1 mm
Step2 soft and transition masks found
skipped cases = 0
test split not used for Ridge-alpha selection
```

### 7. Export and evaluate DFAUST Strict-180-Gap10

DFAUST is used as external registered geometry, not as canonical raw-scan ground truth.

Optional input audit:

```powershell
python check_dfaust_suitability.py `
  --h5_f "<DFAUST>/registrations_f.hdf5" `
  --h5_m "<DFAUST>/registrations_m.hdf5" `
  --out_dir "<DFAUST_AUDIT>"
```

Export the fixed external set:

```powershell
python export_dfaust_balanced_large_scale.py `
  --female_h5 "<DFAUST>/registrations_f.hdf5" `
  --male_h5 "<DFAUST>/registrations_m.hdf5" `
  --step2_root "<STEP2_SUPPORT>" `
  --output_dir "<RESULT_ROOT>/DFAUST/cafer_external_cases_dfaust_strict180_gap10"
```

The default strict protocol:

1. scans every fifth registered frame;
2. uses translation-only centroid alignment;
3. retains frames with centroid-aligned template MPVPE at or below 180 mm;
4. enforces a minimum frame-index gap of 10 within each subject-sequence;
5. exports every eligible frame without subject, sequence, or gender caps.

With the paper data this yields 108 cases from nine subjects and 25 subject-sequences. Do not select cases again based on reconstruction errors.

Run the separate final weighted external evaluation:

```powershell
python run_cafer_external_registered_targets.py `
  --external_cases_dir "<RESULT_ROOT>/DFAUST/cafer_external_cases_dfaust_strict180_gap10" `
  --output_dir "<RESULT_ROOT>/DFAUST/external_registered_targets_dfaust_strict180_gap10" `
  --step2_root "<STEP2_SUPPORT>" `
  --smpl_model_dir "<SMPL_MODEL_DIR>"
```

Do not use the skip-method or incomplete-result flags for a formal run.

### 8. Run the external equal-capacity Table B

After the Step3 fair-capacity experiment and the weighted external evaluation exist, run:

```powershell
python run_dfaust_equal_dim_610.py
```

This script requires no command-line arguments. It reads `DIGITAL_MAN_BASE_DIR`, reuses the exact Step3 fair model definitions, basis normalization, cached solver construction, and fixed per-model Ridge, and evaluates all 108 fixed cases in four batches.

The script intentionally fails if it cannot recover the exact Step3 models or Ridge. It does not construct substitute bases, pad a rank-deficient basis, tune Ridge on DFAUST, skip cases, or mix the weighted CaFeR result into Table B.

The output directory is:

```text
<RESULT_ROOT>/DFAUST/external_equal_capacity_610_unweighted_strict180_gap10/
```

A complete run must satisfy:

```json
{
  "passed": true,
  "checks": {
    "four_methods_present": true,
    "all_models_are_610D": true,
    "expected_432_rows": true,
    "same_108_cases_for_every_method": true,
    "no_duplicate_method_case": true,
    "all_basis_rank_610": true,
    "no_failed_cases": true,
    "unweighted_fit_protocol": true,
    "no_manual_ridge_fallback": true,
    "ridge_inherited_from_step3_cache": true,
    "four_step3_cached_cholesky_solves": true
  }
}
```

Inspect `run_error.txt` if the run fails. Do not treat partial CSV files as completed results.

## Reference results

These values are included to make reproduction drift visible. All errors are lower-is-better.

### Controlled synthetic capacity benchmark

The 1,000-target comparison uses the unweighted fitting protocol. Direct beta is a 10D reference and is not part of the 610D capacity ranking.

| Method | Dim. | MPVPE (mm) | Transition (mm) | Boundary residual jump (mm) | Normal error (deg.) |
|---|---:|---:|---:|---:|---:|
| Direct SMPL beta | 10 | 2.395 | 2.348 | 2.469 | 1.254 |
| Global spectral | 610 | 1.285 | 1.712 | 1.890 | 1.162 |
| Hard-mask local spectral | 610 | 0.284 | 0.653 | 1.011 | 0.791 |
| Soft-mask local spectral | 610 | 0.376 | 0.648 | 0.997 | 0.866 |
| **CaFeR Linear-610** | **610** | **0.234** | **0.434** | **0.567** | **0.622** |

### Shared-LBS diagnostic

For 1,000 cases and seven shared poses, the deployable `before_high` Linear-610 reconstruction has a mean rest-space MPVPE of 0.230369 mm, a posed-space MPVPE of 0.238922 mm, and a mean posed/rest ratio of 1.035728.

### Sparse registered observation diagnostic

For 512 registered canonical vertices, the Ridge predictor has a test prediction-to-solver MPVPE of 0.014605 mm and an additional target error gap of 0.004649 mm. This experiment assumes known sparse vertex identities.

### DFAUST final weighted external validation (Table A)

| Method | Full MPVPE (mm) | Body-core (mm) | Extremity (mm) | Transition (mm) | Boundary residual jump (mm) |
|---|---:|---:|---:|---:|---:|
| SMPL beta + translation | 99.14 | 84.59 | 135.58 | 103.13 | 13.03 |
| Direct beta | 99.62 | 85.28 | 135.53 | 100.17 | 13.60 |
| **CaFeR Linear-610, final weighted fit** | **16.56** | **11.67** | **28.79** | **30.84** | 40.04 |
| CaFeR + Oracle-Transition | 14.08 | 9.48 | 25.59 | 6.31 | 0.68 |

The 16.56 mm result uses the frozen final weighted CaFeR solver. Oracle-Transition is target-derived and diagnostic only.

### DFAUST equal-capacity unweighted comparison (Table B)

| Method | Dim. | Full MPVPE (mm) | Transition (mm) | Boundary residual jump (mm) | Normal error (deg.) |
|---|---:|---:|---:|---:|---:|
| **Global spectral** | **610** | **8.175** | **10.136** | **8.427** | **24.354** |
| Hard-mask local spectral | 610 | 10.365 | 12.772 | 19.425 | 27.368 |
| CaFeR Linear-610 | 610 | 16.684 | 27.702 | 37.231 | 34.074 |
| Soft-mask local spectral | 610 | 25.008 | 38.932 | 56.211 | 35.801 |

The verified run contains four models, 108 cases, and 432 unique method-case rows; every basis has numerical rank 610, all methods use the same cases, and no manual Ridge fallback is used. CaFeR improves over the soft-mask local spectral representation but does not outperform the global or hard-mask spectral bases on these dynamic registered residuals.

## Metrics and statistics

The main evaluation code reports:

- full-body MPVPE;
- body-core and extremity MPVPE;
- transition-region MPVPE;
- six-region MPVPE;
- boundary residual jump, mean and 95th percentile;
- normal error;
- target/non-target region errors and leakage diagnostics;
- subject, sequence, gender, and distance-group summaries where applicable.

The DFAUST experiments use subject-level macro summaries and subject-clustered bootstrap confidence intervals because frames from one subject are correlated. Pairwise differences are defined as

```text
baseline error - CaFeR error
```

so a positive value means lower CaFeR error.

## Visualization utilities

- `plot_figure2_six_region_soft_support.py` renders existing Step1/Step2 support assets; it does not recompute Step2.
- `fig_dimension.py` plots MPVPE and boundary residual jump versus dimension from the fair-baseline CSV files.
- `vis1.py` renders controlled-target reconstruction and Oracle-Transition diagnostics.
- `visualize_lbs_pose_compatibility.py` renders selected shared-LBS poses.
- `make_dfaust_qualitative_figure_representative.py` selects quantile-based DFAUST cases with subject/sequence diversity and is preferred for representative figures.
- `make_dfaust_qualitative_figure.py` selects lowest-error cases and should not be used as evidence of representative performance.
- `make_step5_sparse_count_sweep_figure.py` should be pointed to valid Step5 CSV outputs; it contains a fallback result table for plotting convenience, so always verify the data source recorded by the script.

Visualization-only smoothing or Oracle residuals do not alter quantitative tables and must be labeled explicitly.

## Known implementation limitations

1. Several source files still contain legacy absolute Windows defaults. Configure or override paths before running, and remove such paths from any public or anonymous release artifact.
2. `step2_validate_results.py` currently expects a `num_vertices` column, while the current Step2 writer stores `hard_vertex_count`. This can cause a `KeyError` during the region-count check. Do not claim automated Step2 validation passed unless the report completed and was inspected.
3. Some loaders permit fallback assets, such as zero deformation sensitivity or labels derived from `argmax(soft)`. These fallbacks are convenient for debugging but do not constitute exact paper reproduction.
4. The fair synthetic runner checks requested dimensions but does not itself perform a full basis-rank test. The formal external 610D runner performs the required rank-610 verification.
5. Step4 checks the 6,890-by-3 vertex shape but does not independently hash every input topology. Preserve the verified Step3/Step2 vertex order.
6. Step5 can skip invalid cases and continue. A formal result requires `skipped cases = 0` and the exact explicit Step3 decoder.
7. `step3/make_step3_qualitative_grid.py` has a colorbar-unit labeling risk for arrays stored in meters. Verify units before using that figure in a paper.
8. No project-level software license is included in this snapshot. The presence of source code does not grant rights to redistribute SMPL, DFAUST, AMASS, or generated assets derived from restricted data.

## Anonymous release checklist

Before uploading a code or supplementary archive for double-blind review:

- replace or remove all personal absolute paths in source, CSV, JSON, and manifests;
- remove `__pycache__`, `.pyc`, zero-byte helper files, nested old ZIPs, and legacy result directories;
- do not include SMPL model files, DFAUST registrations, AMASS files, or other licensed assets;
- remove filesystem paths from Step4 tables, Step5 sample indexes, decoder manifests, and external case metadata;
- include only the current outer `step1/` through `step5/` sources;
- keep `step1_multi/` and `step3-1/` out of the canonical reproduction instructions;
- preserve anonymous paper and repository metadata until the review process is complete.

## Citation

This is an anonymous submission. Citation metadata will be added after the review process.
